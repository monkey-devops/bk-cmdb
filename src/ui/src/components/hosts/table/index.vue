<template>
    <div class="hosts-table-layout">
        <div class="hosts-options">
            <div class="options-left">
                <slot name="options-left">
                    <bk-button class="options-button mr10" theme="primary"
                        :disabled="!table.checked.length"
                        @click="handleMultipleEdit">
                        {{$t('编辑')}}
                    </bk-button>
                    <span class="inline-block-middle mr10"
                        v-cursor="{
                            active: !$isAuthorized(transferAuth),
                            auth: [transferAuth]
                        }">
                        <bk-button class="options-button" theme="default"
                            :disabled="!table.checked.length || !$isAuthorized(transferAuth)"
                            @click="transfer.show = true">
                            {{$t('转移')}}
                        </bk-button>
                    </span>
                    <bk-button class="options-button mr10"
                        theme="default"
                        type="submit"
                        form="exportForm"
                        :disabled="!table.checked.length">
                        {{$t('导出')}}
                    </bk-button>
                    <form id="exportForm" :action="table.exportUrl" method="POST" hidden>
                        <input type="hidden" name="bk_host_id" :value="table.checked">
                        <input type="hidden" name="export_custom_fields"
                            v-if="usercustom[columnsConfigKey]"
                            :value="usercustom[columnsConfigKey]">
                        <input type="hidden" name="bk_biz_id" value="-1">
                        <input type="hidden" name="metadata"
                            v-if="$route.name !== 'resource'"
                            :value="JSON.stringify($injectMetadata().metadata)">
                    </form>
                    <cmdb-clipboard-selector class="options-button"
                        :list="clipboardList"
                        :disabled="!table.checked.length"
                        @on-copy="handleCopy">
                    </cmdb-clipboard-selector>
                </slot>
            </div>
            <div class="options-right clearfix">
                <div class="fl" v-if="showScope">
                    <i class="options-split"></i>
                    <bk-select class="options-collection"
                        v-model="scope"
                        :clearable="false">
                        <bk-option id="all" :name="$t('全部主机')"></bk-option>
                        <bk-option :id="0" :name="$t('已分配主机')"></bk-option>
                        <bk-option :id="1" :name="$t('未分配主机')"></bk-option>
                    </bk-select>
                </div>
                <div class="fr">
                    <bk-select class="options-collection"
                        v-if="showCollection"
                        ref="collectionSelector"
                        v-model="selectedCollection"
                        :loading="$loading('searchCollection')"
                        :placeholder="$t('请选择收藏条件')"
                        @change="handleSelectCollection">
                        <bk-option v-for="collection in collectionList"
                            :key="collection.id"
                            :id="collection.id"
                            :name="collection.name">
                        </bk-option>
                        <div slot="extension">
                            <a href="javascript:void(0)" class="collection-create" @click="handleCreateCollection">
                                <i class="bk-icon icon-plus-circle"></i>
                                {{$t('新增条件')}}
                            </a>
                        </div>
                    </bk-select>
                    <cmdb-host-filter class="ml10"
                        ref="hostFilter"
                        :properties="filterProperties"
                        :show-scope="showScope">
                    </cmdb-host-filter>
                    <bk-button class="options-button icon-btn ml10"
                        icon="icon-cc-setting"
                        v-bk-tooltips.top="$t('列表显示属性配置')"
                        @click="handleColumnConfigClick">
                    </bk-button>
                    <bk-button class="options-button icon-btn ml10" v-if="showHistory"
                        v-bk-tooltips="$t('查看删除历史')"
                        icon="icon-cc-history"
                        @click="routeToHistory">
                    </bk-button>
                </div>
            </div>
        </div>
        <bk-table class="hosts-table"
            v-bkloading="{ isLoading: $loading() }"
            :data="table.list"
            :pagination="table.pagination"
            :max-height="$APP.height - 150"
            @selection-change="handleSelectionChange"
            @row-click="handleRowClick"
            @sort-change="handleSortChange"
            @page-change="handlePageChange"
            @page-limit-change="handleSizeChange">
            <bk-table-column type="selection" width="60" align="center" fixed class-name="bk-table-selection"></bk-table-column>
            <bk-table-column v-for="column in table.header"
                :key="column.id"
                :label="column.name"
                :sortable="column.sortable ? 'custom' : false"
                :prop="column.id"
                :fixed="column.id === 'bk_host_innerip'"
                :class-name="column.id === 'bk_host_innerip' ? 'is-highlight' : ''">
                <template slot-scope="{ row }">
                    {{ row | hostValueFilter(column.objId, column.id) | formatter(column.type) }}
                </template>
            </bk-table-column>
        </bk-table>
        <bk-sideslider
            :is-show.sync="slider.show"
            :title="slider.title"
            :width="800"
            :before-close="handleSliderBeforeClose">
            <bk-tab :active.sync="tab.active" type="unborder-card" slot="content" v-if="slider.show">
                <bk-tab-panel name="attribute" :label="$t('属性')" style="width: calc(100% + 40px);margin: 0 -20px;">
                    <cmdb-form-multiple v-if="tab.attribute.type === 'multiple'"
                        ref="multipleForm"
                        :properties="properties.host"
                        :property-groups="propertyGroups"
                        :object-unique="objectUnique"
                        @on-submit="handleMultipleSave"
                        @on-cancel="handleSliderBeforeClose">
                    </cmdb-form-multiple>
                </bk-tab-panel>
            </bk-tab>
        </bk-sideslider>
        <bk-sideslider
            :is-show.sync="columnsConfig.show"
            :width="600"
            :title="$t('列表显示属性配置')">
            <cmdb-columns-config slot="content"
                v-if="columnsConfig.show"
                :properties="columnsConfigProperties"
                :selected="columnsConfig.selected"
                :disabled-columns="columnsConfig.disabledColumns"
                @on-apply="handleApplyColumnsConfig"
                @on-cancel="columnsConfig.show = false"
                @on-reset="handleResetColumnsConfig">
            </cmdb-columns-config>
        </bk-sideslider>
        <bk-dialog class="bk-dialog-no-padding"
            v-model="transfer.show"
            draggable
            :close-icon="false"
            :show-footer="false"
            :show-header="false"
            :width="720">
            <div class="transfer-title" slot="tools">
                <i class="icon icon-cc-shift mr5"></i>
                <span>{{$t('主机转移')}}</span>
                <span v-if="selectedHosts.length === 1">{{selectedHosts[0]['host']['bk_host_innerip']}}</span>
            </div>
            <div class="transfer-content">
                <cmdb-transfer-host v-if="transfer.show"
                    :transfer-resource-auth="transferResourceAuth"
                    :selected-hosts="selectedHosts"
                    @on-success="handleTransferSuccess"
                    @on-cancel="transfer.show = false">
                </cmdb-transfer-host>
            </div>
        </bk-dialog>
    </div>
</template>

<script>
    import { mapGetters, mapActions, mapState } from 'vuex'
    import cmdbColumnsConfig from '@/components/columns-config/columns-config'
    import cmdbTransferHost from '@/components/hosts/transfer'
    import cmdbHostFilter from '@/components/hosts/filter/index.vue'
    import hostValueFilter from '@/filters/host'
    export default {
        components: {
            cmdbColumnsConfig,
            cmdbTransferHost,
            cmdbHostFilter
        },
        filters: {
            hostValueFilter
        },
        props: {
            columnsConfigProperties: {
                type: Array,
                required: true
            },
            columnsConfigKey: {
                type: String,
                required: true
            },
            columnsConfigDisabledColumns: {
                type: Array,
                default () {
                    return ['bk_host_innerip', 'bk_cloud_id', 'bk_module_name']
                }
            },
            saveAuth: {
                type: [String, Array],
                default: ''
            },
            editAuth: {
                type: [String, Array],
                default: ''
            },
            deleteAuth: {
                type: [String, Array],
                default: ''
            },
            transferAuth: {
                type: [String, Array],
                default: ''
            },
            transferResourceAuth: {
                type: [String, Array],
                default: ''
            },
            showCollection: Boolean,
            showHistory: Boolean,
            showScope: Boolean
        },
        data () {
            return {
                objectUnique: [],
                properties: {
                    biz: [],
                    host: [],
                    set: [],
                    module: []
                },
                propertyGroups: [],
                table: {
                    checked: [],
                    header: [],
                    list: [],
                    pagination: {
                        current: 1,
                        limit: 10,
                        count: 0,
                        limitList: [10, 50, 100, 500]
                    },
                    defaultSort: 'bk_host_id',
                    sort: 'bk_host_id',
                    exportUrl: `${window.API_HOST}hosts/export`,
                    tableMinusHeight: 200
                },
                filter: {
                    business: '',
                    condition: {}
                },
                slider: {
                    show: false,
                    title: ''
                },
                tab: {
                    active: 'attribute',
                    attribute: {
                        type: 'details',
                        inst: {
                            details: {},
                            edit: {},
                            original: {}
                        }
                    }
                },
                columnsConfig: {
                    show: false,
                    selected: [],
                    disabledColumns: ['bk_host_innerip', 'bk_cloud_id', 'bk_module_name', 'bk_set_name']
                },
                transfer: {
                    show: false
                },
                selectedCollection: '',
                scope: 'all'
            }
        },
        computed: {
            ...mapGetters(['supplierAccount']),
            ...mapGetters('userCustom', ['usercustom']),
            ...mapState('hosts', ['collectionList']),
            customColumns () {
                return this.usercustom[this.columnsConfigKey] || []
            },
            clipboardList () {
                return this.table.header.filter(header => header.type !== 'checkbox')
            },
            selectedHosts () {
                return this.table.list.filter(host => this.table.checked.includes(host['host']['bk_host_id']))
            },
            filterProperties () {
                const { module, set, host } = this.properties
                const filterProperty = ['bk_host_innerip', 'bk_host_outerip', 'bk_cloud_id']
                return {
                    host: host.filter(property => !filterProperty.includes(property.bk_property_id)),
                    module,
                    set
                }
            }
        },
        watch: {
            'table.checked' (checked) {
                this.$emit('on-checked', checked)
            },
            'table.header' (header) {
                this.$emit('on-set-header', header)
            },
            'slider.show' (show) {
                if (!show) {
                    this.tab.active = 'attribute'
                }
            },
            customColumns () {
                this.setTableHeader()
            },
            columnsConfigProperties () {
                this.setTableHeader()
            },
            scope () {
                this.handlePageChange(1)
            }
        },
        async created () {
            try {
                await Promise.all([
                    this.getProperties(),
                    this.getHostPropertyGroups()
                ])
                if (this.showCollection) {
                    this.getCollectionList()
                }
            } catch (e) {
                console.log(e)
            }
        },
        methods: {
            ...mapActions('objectModelProperty', ['batchSearchObjectAttribute']),
            ...mapActions('objectModelFieldGroup', ['searchGroup']),
            ...mapActions('hostUpdate', ['updateHost']),
            ...mapActions('hostSearch', ['searchHost', 'searchHostByInnerip']),
            getProperties () {
                return this.batchSearchObjectAttribute({
                    params: this.$injectMetadata({
                        bk_obj_id: { '$in': Object.keys(this.properties) },
                        bk_supplier_account: this.supplierAccount
                    }, { inject: this.$route.name !== 'resource' }),
                    config: {
                        requestId: `post_batchSearchObjectAttribute_${Object.keys(this.properties).join('_')}`,
                        requestGroup: Object.keys(this.properties).map(id => `post_searchObjectAttribute_${id}`)
                    }
                }).then(result => {
                    Object.keys(this.properties).forEach(objId => {
                        this.properties[objId] = result[objId]
                    })
                    return result
                })
            },
            getHostPropertyGroups () {
                return this.searchGroup({
                    objId: 'host',
                    params: this.$injectMetadata(),
                    config: {
                        fromCache: true,
                        requestId: 'post_searchGroup_host'
                    }
                }).then(groups => {
                    this.propertyGroups = groups
                    return groups
                })
            },
            async getCollectionList () {
                try {
                    const data = await this.$store.dispatch('hostFavorites/searchFavorites', {
                        params: {},
                        config: {
                            requestId: 'searchCollection'
                        }
                    })
                    this.$store.commit('hosts/setCollectionList', data.info)
                } catch (e) {
                    console.error(e)
                }
            },
            handleSelectCollection (value) {
                if (value) {
                    const collection = this.collectionList.find(collection => collection.id === value)
                    try {
                        const filterList = JSON.parse(collection.query_params).map(condition => {
                            return {
                                bk_obj_id: condition.bk_obj_id,
                                bk_property_id: condition.field,
                                operator: condition.operator,
                                value: condition.value
                            }
                        })
                        const info = JSON.parse(collection.info)
                        const filterIP = {
                            text: info.ip_list.join('\n'),
                            exact: info.exact_search,
                            inner: info.bk_host_innerip,
                            outer: info.bk_host_outerip
                        }
                        this.$store.commit('hosts/setFilterList', filterList)
                        this.$store.commit('hosts/setFilterIP', filterIP)
                        this.$store.commit('hosts/setCollection', collection)
                        setTimeout(() => {
                            this.$refs.hostFilter.handleSearch(false)
                        }, 0)
                    } catch (e) {
                        this.$error(this.$t('应用收藏条件失败，转换数据错误'))
                        console.error(e.message)
                    }
                } else {
                    this.$refs.hostFilter.handleReset()
                    const key = this.$route.meta.filterPropertyKey
                    const customData = this.$store.getters['userCustom/getCustomData'](key, [])
                    this.$store.commit('hosts/setFilterList', customData)
                }
            },
            handleCreateCollection () {
                this.selectedCollection = ''
                this.$refs.collectionSelector.close()
                this.$refs.hostFilter.handleToggleFilter()
            },
            setTableHeader () {
                const properties = this.$tools.getHeaderProperties(this.columnsConfigProperties, this.customColumns, this.columnsConfigDisabledColumns)
                this.table.header = properties.map(property => {
                    return {
                        id: property.bk_property_id,
                        name: property.bk_property_name,
                        type: property.bk_property_type,
                        objId: property.bk_obj_id,
                        sortable: property.bk_obj_id === 'host' && !['foreignkey'].includes(property.bk_property_type)
                    }
                })
                this.columnsConfig.selected = properties.map(property => property['bk_property_id'])
            },
            getHostCellText (header, item) {
                const objId = header.objId
                const propertyId = header.id
                const headerProperty = this.$tools.getProperty(this.properties[objId], propertyId)
                const originalValues = item[objId] instanceof Array ? item[objId] : [item[objId]]
                const text = []
                originalValues.forEach(value => {
                    const flattenedText = this.$tools.getPropertyText(headerProperty, value)
                    flattenedText ? text.push(flattenedText) : void (0)
                })
                return text.join(',') || '--'
            },
            getHostList () {
                this.searchHost({
                    params: this.injectScope({
                        ...this.filter.condition,
                        'bk_biz_id': this.filter.business,
                        page: {
                            start: (this.table.pagination.current - 1) * this.table.pagination.limit,
                            limit: this.table.pagination.limit,
                            sort: this.table.sort
                        }
                    }),
                    config: {
                        requestId: 'searchHosts',
                        cancelPrevious: true
                    }
                }).then(data => {
                    this.table.pagination.count = data.count
                    this.table.list = data.info
                    return data
                }).catch(e => {
                    this.table.checked = []
                    this.table.list = []
                    this.table.pagination.count = 0
                })
            },
            injectScope (params) {
                const biz = params.condition.find(condition => condition.bk_obj_id === 'biz')
                if (this.scope === 'all') {
                    biz.condition = biz.condition.filter(condition => condition.field !== 'default')
                } else {
                    const newCondition = {
                        field: 'default',
                        operator: '$eq',
                        value: this.scope
                    }
                    const existCondition = biz.condition.find(condition => condition.field === 'default')
                    if (existCondition) {
                        Object.assign(existCondition, newCondition)
                    } else {
                        biz.condition.push(newCondition)
                    }
                }
                return params
            },
            search (business, condition, resetPage = false) {
                this.filter.business = business
                this.filter.condition = condition
                if (resetPage) {
                    this.table.pagination.current = 1
                }
                this.getHostList()
            },
            handlePageChange (current) {
                this.table.pagination.current = current
                this.getHostList()
            },
            handleSizeChange (limit) {
                this.table.pagination.limit = limit
                this.handlePageChange(1)
            },
            handleSortChange (sort) {
                this.table.sort = this.$tools.getSort(sort)
                this.handlePageChange(1)
            },
            handleCopy (target) {
                const copyList = this.table.list.filter(item => {
                    return this.table.checked.includes(item['host']['bk_host_id'])
                })
                const copyText = []
                this.$tools.clone(copyList).forEach(item => {
                    const cellText = this.getHostCellText(target, item)
                    if (cellText !== '--') {
                        copyText.push(cellText)
                    }
                })
                if (copyText.length) {
                    this.$copyText(copyText.join('\n')).then(() => {
                        this.$success(this.$t('复制成功'))
                    }, () => {
                        this.$error(this.$t('复制失败'))
                    })
                } else {
                    this.$info(this.$t('该字段无可复制的值'))
                }
            },
            handleSelectionChange (selection) {
                this.table.checked = selection.map(item => item.host.bk_host_id)
            },
            handleRowClick (item) {
                const business = item.biz[0]
                if (!business.default) {
                    this.$router.push({
                        name: 'businessHostDetails',
                        params: {
                            business: business.bk_biz_id,
                            id: item.host.bk_host_id
                        },
                        query: {
                            from: this.$route.fullPath
                        }
                    })
                } else {
                    this.$router.push({
                        name: 'resourceHostDetails',
                        params: {
                            id: item.host.bk_host_id
                        },
                        query: {
                            from: this.$route.fullPath
                        }
                    })
                }
            },
            batchUpdate (params) {
                return this.updateHost({ params }).then(data => {
                    this.$success(this.$t('保存成功'))
                    this.getHostList()
                    return data
                })
            },
            async handleMultipleEdit () {
                this.objectUnique = await this.$store.dispatch('objectUnique/searchObjectUniqueConstraints', {
                    objId: 'host',
                    params: this.$injectMetadata({}, {
                        inject: this.$route.name !== 'resource'
                    })
                })
                this.tab.attribute.type = 'multiple'
                this.slider.title = this.$t('主机属性')
                this.slider.show = true
            },
            async handleMultipleSave (changedValues) {
                await this.batchUpdate(this.$injectMetadata({
                    ...changedValues,
                    'bk_host_id': this.table.checked.join(',')
                }, { inject: this.$route.name !== 'resource' }))
                this.slider.show = false
            },
            handleMultipleCancel () {
                this.slider.show = false
            },
            handleApplyColumnsConfig (properties) {
                this.$store.dispatch('userCustom/saveUsercustom', {
                    [this.columnsConfigKey]: properties.map(property => property['bk_property_id'])
                })
                this.columnsConfig.show = false
            },
            handleResetColumnsConfig () {
                this.$store.dispatch('userCustom/saveUsercustom', {
                    [this.columnsConfigKey]: []
                })
            },
            handleTransferSuccess () {
                this.table.checked = []
                this.transfer.show = false
                this.getHostList()
            },
            handleSliderBeforeClose () {
                if (this.tab.active === 'attribute' && this.tab.attribute.type !== 'details') {
                    const $form = this.tab.attribute.type === 'update' ? this.$refs.form : this.$refs.multipleForm
                    const changedValues = $form.changedValues
                    if (Object.keys(changedValues).length) {
                        return new Promise((resolve, reject) => {
                            this.$bkInfo({
                                title: this.$t('确认退出'),
                                subTitle: this.$t('退出会导致未保存信息丢失'),
                                extCls: 'bk-dialog-sub-header-center',
                                confirmFn: () => {
                                    this.slider.show = false
                                },
                                cancelFn: () => {
                                    resolve(false)
                                }
                            })
                        })
                    }
                    this.slider.show = false
                }
                this.slider.show = false
            },
            handleQuickSearch (property, value, operator) {
                this.$emit('on-quick-search', property, value, operator)
            },
            routeToHistory () {
                this.$router.push({
                    name: 'history',
                    params: {
                        objId: 'host'
                    },
                    query: {
                        from: this.$route.fullPath
                    }
                })
            },
            handleColumnConfigClick () {
                this.$refs.hostFilter.$refs.filterPopper.instance.hide()
                this.columnsConfig.show = true
            }
        }
    }
</script>

<style lang="scss" scoped>
    .hosts-options{
        font-size: 0;
        .options-left {
            float: left;
        }
        .options-right {
            overflow: hidden;
            .options-split {
                @include inlineBlock;
                width: 2px;
                height: 20px;
                margin: 0 10px;
                background-color: #DCDEE5;
            }
        }
        .icon-btn {
            width: 32px;
            padding: 0;
            line-height: 14px;
        }
        .options-button{
            position: relative;
            display: inline-block;
            vertical-align: middle;
            font-size: 14px;
            &.quick-search-button {
                .icon-angle-down {
                    font-size: 12px;
                    top: 0;
                }
            }
            &:first-child {
                margin-left: 0;
            }
            &:hover{
                z-index: 1;
            }
        }
    }
    .hosts-table {
        margin-top: 14px;
    }
    .transfer-title {
        height: 50px;
        line-height: 50px;
        background-color: #f9f9f9;
        color: #333948;
        font-weight: bold;
        font-size: 14px;
        padding: 0 30px;
        border-bottom: 1px solid $cmdbBorderColor;
    }
    .transfer-content {
        height: 540px;
    }
    .options-collection {
        width: 200px;
    }
    .collection-create {
        display: inline-block;
        width: 60%;
        font-size: 12px;
        color: #63656E;
        line-height: 32px;
        cursor: pointer;
        &:hover {
            color: #3a84ff;
        }
        .bk-icon {
            font-size: 14px;
            display: inline-block;
            vertical-align: -2px;
        }
    }
</style>

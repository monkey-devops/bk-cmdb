<template>
    <div class="business-layout">
        <div class="business-options clearfix">
            <span class="fl" v-if="isAdminView"
                v-cursor="{
                    active: !$isAuthorized($OPERATION.C_BUSINESS),
                    auth: [$OPERATION.C_BUSINESS]
                }">
                <bk-button class="fl" theme="primary"
                    :disabled="!$isAuthorized($OPERATION.C_BUSINESS)"
                    @click="handleCreate">
                    {{$t('新建')}}
                </bk-button>
            </span>
            <div class="options-button fr">
                <span class="inline-block-middle" v-cursor="{
                    active: !$isAuthorized($OPERATION.BUSINESS_ARCHIVE),
                    auth: [$OPERATION.BUSINESS_ARCHIVE]
                }">
                    <bk-button class="button-history"
                        icon="icon-cc-history"
                        v-bk-tooltips.bottom="$t('查看已归档业务')"
                        :disabled="!$isAuthorized($OPERATION.BUSINESS_ARCHIVE)"
                        @click="routeToHistory">
                    </bk-button>
                </span>
                <bk-button class="button-setting"
                    icon="icon-cc-setting"
                    v-bk-tooltips.bottom="$t('列表显示属性配置')"
                    @click="columnsConfig.show = true">
                </bk-button>
            </div>
            <div class="options-filter clearfix fr">
                <bk-select
                    class="filter-selector fl"
                    v-model="filter.id"
                    searchable
                    :clearable="false">
                    <bk-option v-for="(option, index) in filter.options"
                        :key="index"
                        :id="option.id"
                        :name="option.name">
                    </bk-option>
                </bk-select>
                <cmdb-form-enum class="filter-value fl"
                    v-if="filter.type === 'enum'"
                    :options="$tools.getEnumOptions(properties, filter.id)"
                    :allow-clear="true"
                    v-model="filter.value"
                    @on-selected="handleFilterData">
                </cmdb-form-enum>
                <bk-input class="filter-value cmdb-form-input fl" type="text" maxlength="11"
                    v-else-if="filter.type === 'int'"
                    v-model.number="filter.value"
                    :placeholder="$t('快速查询')"
                    @enter="handleFilterData">
                </bk-input>
                <bk-input class="filter-value cmdb-form-input fl" type="text"
                    v-else
                    v-model.trim="filter.value"
                    :placeholder="$t('快速查询')"
                    @enter="handleFilterData">
                </bk-input>
                <i class="filter-search bk-icon icon-search"
                    v-show="filter.type !== 'enum'"
                    @click="handleFilterData"></i>
            </div>
        </div>
        <bk-table class="business-table"
            v-bkloading="{ isLoading: $loading('post_searchBusiness_list') }"
            :data="table.list"
            :pagination="table.pagination"
            :max-height="$APP.height - 160"
            @row-click="handleRowClick"
            @sort-change="handleSortChange"
            @page-limit-change="handleSizeChange"
            @page-change="handlePageChange">
            <bk-table-column prop="bk_biz_id" label="ID" width="50" align="center" fixed></bk-table-column>
            <bk-table-column v-for="column in table.header"
                :class-name="column.id === 'bk_biz_name' ? 'is-highlight' : ''"
                sortable="custom"
                :fixed="column.id === 'bk_biz_name'"
                :key="column.id"
                :prop="column.id"
                :label="column.name">
            </bk-table-column>
        </bk-table>
        <bk-sideslider
            :is-show.sync="slider.show"
            :title="slider.title"
            :width="800"
            :before-close="handleSliderBeforeClose">
            <bk-tab :active.sync="tab.active" type="unborder-card" slot="content" v-if="slider.show">
                <bk-tab-panel name="attribute" :label="$t('属性')" style="width: calc(100% + 40px);margin: 0 -20px;">
                    <cmdb-details v-if="attribute.type === 'details'"
                        :properties="properties"
                        :property-groups="propertyGroups"
                        :inst="attribute.inst.details"
                        :delete-button-text="$t('归档')"
                        :show-delete="attribute.inst.details['bk_biz_name'] !== '蓝鲸'"
                        :show-options="isAdminView"
                        :edit-auth="$OPERATION.U_BUSINESS"
                        :delete-auth="$OPERATION.BUSINESS_ARCHIVE"
                        @on-edit="handleEdit"
                        @on-delete="handleDelete">
                    </cmdb-details>
                    <cmdb-form v-else-if="['update', 'create'].includes(attribute.type)"
                        ref="form"
                        :properties="properties"
                        :property-groups="propertyGroups"
                        :inst="attribute.inst.edit"
                        :type="attribute.type"
                        :save-auth="saveAuth"
                        @on-submit="handleSave"
                        @on-cancel="handleSliderBeforeClose">
                    </cmdb-form>
                </bk-tab-panel>
                <bk-tab-panel name="relevance" :label="$t('关联Relation')" :visible="attribute.type !== 'create'">
                    <cmdb-relation
                        v-if="tab.active === 'relevance'"
                        obj-id="biz"
                        :auth="$OPERATION.U_BUSINESS"
                        :inst="attribute.inst.details">
                    </cmdb-relation>
                </bk-tab-panel>
                <bk-tab-panel name="history" :label="$t('变更记录')" :visible="attribute.type !== 'create'">
                    <cmdb-audit-history v-if="tab.active === 'history'"
                        target="biz"
                        :inst-id="attribute.inst.details['bk_biz_id']">
                    </cmdb-audit-history>
                </bk-tab-panel>
            </bk-tab>
        </bk-sideslider>
        <bk-sideslider :is-show.sync="columnsConfig.show" :width="600" :title="$t('列表显示属性配置')">
            <cmdb-columns-config slot="content"
                v-if="columnsConfig.show"
                :properties="properties"
                :selected="columnsConfig.selected"
                :disabled-columns="columnsConfig.disabledColumns"
                @on-apply="handleApplayColumnsConfig"
                @on-cancel="columnsConfig.show = false"
                @on-reset="handleResetColumnsConfig">
            </cmdb-columns-config>
        </bk-sideslider>
    </div>
</template>

<script>
    import { mapGetters, mapActions } from 'vuex'
    import cmdbColumnsConfig from '@/components/columns-config/columns-config'
    import cmdbAuditHistory from '@/components/audit-history/audit-history.vue'
    import cmdbRelation from '@/components/relation'
    export default {
        components: {
            cmdbColumnsConfig,
            cmdbAuditHistory,
            cmdbRelation
        },
        data () {
            return {
                properties: [],
                propertyGroups: [],
                table: {
                    header: [],
                    list: [],
                    pagination: {
                        count: 0,
                        limit: 10,
                        current: 1
                    },
                    defaultSort: 'bk_biz_id',
                    sort: 'bk_biz_id'
                },
                filter: {
                    id: '',
                    value: '',
                    type: '',
                    options: [],
                    sendValue: ''
                },
                slider: {
                    show: false,
                    title: ''
                },
                tab: {
                    active: 'attribute'
                },
                attribute: {
                    type: null,
                    inst: {
                        edit: {},
                        details: {}
                    }
                },
                columnsConfig: {
                    show: false,
                    selected: [],
                    disabledColumns: ['bk_biz_name']
                }
            }
        },
        computed: {
            ...mapGetters(['supplierAccount', 'userName', 'isAdminView']),
            ...mapGetters('userCustom', ['usercustom']),
            ...mapGetters('objectBiz', ['bizId']),
            ...mapGetters('objectModelClassify', ['getModelById']),
            columnsConfigKey () {
                return `${this.userName}_biz_${this.isAdminView ? 'adminView' : this.bizId}_table_columns`
            },
            customBusinessColumns () {
                return this.usercustom[this.columnsConfigKey] || []
            },
            saveAuth () {
                const type = this.attribute.type
                if (type === 'create') {
                    return this.$OPERATION.C_BUSINESS
                } else if (type === 'update') {
                    return this.$OPERATION.U_BUSINESS
                }
                return ''
            },
            model () {
                return this.getModelById('biz') || {}
            }
        },
        watch: {
            'filter.id' (id) {
                this.filter.value = ''
                this.filter.type = (this.$tools.getProperty(this.properties, id) || {})['bk_property_type']
            },
            'slider.show' (show) {
                if (!show) {
                    this.tab.active = 'attribute'
                }
            },
            customBusinessColumns () {
                this.setTableHeader()
            }
        },
        async created () {
            try {
                this.$store.dispatch('userCustom/setRencentlyData', { id: 'business' })
                this.properties = await this.searchObjectAttribute({
                    params: this.$injectMetadata({
                        bk_obj_id: 'biz',
                        bk_supplier_account: this.supplierAccount
                    }),
                    config: {
                        requestId: 'post_searchObjectAttribute_biz',
                        fromCache: true
                    }
                })
                await Promise.all([
                    this.getPropertyGroups(),
                    this.setTableHeader(),
                    this.setFilterOptions()
                ])
                if (this.$route.params.bizName) {
                    this.filter.sendValue = this.$route.params.bizName
                    this.filter.value = this.$route.params.bizName
                }
                this.getTableData()
            } catch (e) {
                // ignore
            }
        },
        methods: {
            ...mapActions('objectModelFieldGroup', ['searchGroup']),
            ...mapActions('objectModelProperty', ['searchObjectAttribute']),
            ...mapActions('objectBiz', [
                'searchBusiness',
                'archiveBusiness',
                'updateBusiness',
                'createBusiness',
                'searchBusinessById'
            ]),
            getPropertyGroups () {
                return this.searchGroup({
                    objId: 'biz',
                    params: this.$injectMetadata(),
                    config: {
                        fromCache: true,
                        requestId: 'post_searchGroup_biz'
                    }
                }).then(groups => {
                    this.propertyGroups = groups
                    return groups
                })
            },
            setTableHeader () {
                return new Promise((resolve, reject) => {
                    const headerProperties = this.$tools.getHeaderProperties(this.properties, this.customBusinessColumns, this.columnsConfig.disabledColumns)
                    resolve(headerProperties)
                }).then(properties => {
                    this.updateTableHeader(properties)
                    this.columnsConfig.selected = properties.map(property => property['bk_property_id'])
                })
            },
            setFilterOptions () {
                this.filter.options = this.properties
                    .filter(property => !['singleasst', 'multiasst'].includes(property['bk_property_type']))
                    .map(property => {
                        return {
                            id: property['bk_property_id'],
                            name: property['bk_property_name']
                        }
                    })
                this.filter.id = this.filter.options.length ? this.filter.options[0]['id'] : ''
            },
            updateTableHeader (properties) {
                this.table.header = properties.map(property => {
                    return {
                        id: property['bk_property_id'],
                        name: property['bk_property_name']
                    }
                })
            },
            handleRowClick (item) {
                this.slider.show = true
                this.slider.title = item['bk_biz_name']
                this.attribute.inst.details = item
                this.attribute.type = 'details'
            },
            handleSortChange (sort) {
                this.table.sort = this.$tools.getSort(sort)
                this.handlePageChange(1)
            },
            handleSizeChange (size) {
                this.table.pagination.limit = size
                this.handlePageChange(1)
            },
            handlePageChange (page) {
                this.table.pagination.current = page
                this.getTableData()
            },
            getBusinessList (config = { cancelPrevious: true }) {
                return this.searchBusiness({
                    params: this.getSearchParams(),
                    config: Object.assign({ requestId: 'post_searchBusiness_list' }, config)
                })
            },
            handleFilterData () {
                this.table.pagination.current = 1
                this.filter.sendValue = this.filter.value
                this.getTableData()
            },
            getTableData () {
                this.getBusinessList().then(data => {
                    if (data.count && !data.info.length) {
                        this.table.pagination.current -= 1
                        this.getTableData()
                    }
                    this.table.list = this.$tools.flattenList(this.properties, data.info)
                    this.table.pagination.count = data.count
                    return data
                })
            },
            getSearchParams () {
                const params = {
                    condition: {
                        'bk_data_status': { '$ne': 'disabled' }
                    },
                    fields: [],
                    page: {
                        start: this.table.pagination.limit * (this.table.pagination.current - 1),
                        limit: this.table.pagination.limit,
                        sort: this.table.sort
                    }
                }
                if (this.filter.id && String(this.filter.sendValue).length) {
                    const filterType = this.filter.type
                    let filterValue = this.filter.sendValue
                    if (filterType === 'bool') {
                        const convertValue = [true, false].find(bool => bool.toString() === filterValue)
                        filterValue = convertValue === undefined ? filterValue : convertValue
                    } else if (filterType === 'int') {
                        filterValue = isNaN(parseInt(filterValue)) ? filterValue : parseInt(filterValue)
                    }
                    params.condition[this.filter.id] = filterValue
                }
                return params
            },
            async handleEdit (flattenItem) {
                const list = await this.getBusinessList({ fromCache: true })
                const inst = list.info.find(item => item['bk_biz_id'] === flattenItem['bk_biz_id'])
                const bizNameProperty = this.$tools.getProperty(this.properties, 'bk_biz_name')
                bizNameProperty.isreadonly = inst['bk_biz_name'] === '蓝鲸'
                this.attribute.inst.edit = inst
                this.attribute.type = 'update'
            },
            handleCreate () {
                this.attribute.type = 'create'
                this.attribute.inst.edit = {}
                this.slider.show = true
                this.slider.title = `${this.$t('创建')} ${this.model['bk_obj_name']}`
            },
            handleDelete (inst) {
                this.$bkInfo({
                    title: this.$t('确认要归档', { name: inst['bk_biz_name'] }),
                    confirmFn: () => {
                        this.archiveBusiness(inst['bk_biz_id']).then(() => {
                            this.slider.show = false
                            this.$success(this.$t('归档成功'))
                            this.getTableData()
                            this.$http.cancel('post_searchBusiness_$ne_disabled')
                        })
                    }
                })
            },
            handleSave (values, changedValues, originalValues, type) {
                if (type === 'update') {
                    this.updateBusiness({
                        bizId: originalValues['bk_biz_id'],
                        params: values
                    }).then(() => {
                        this.getTableData()
                        this.searchBusinessById({ bizId: originalValues['bk_biz_id'] }).then(item => {
                            this.attribute.inst.details = this.$tools.flattenItem(this.properties, item)
                        })
                        this.handleCancel()
                        this.$success(this.$t('修改成功'))
                        this.$http.cancel('post_searchBusiness_$ne_disabled')
                    })
                } else {
                    this.createBusiness({
                        params: values
                    }).then(() => {
                        this.handlePageChange(1)
                        this.handleCancel()
                        this.$success(this.$t('创建成功'))
                        this.$http.cancel('post_searchBusiness_$ne_disabled')
                    })
                }
            },
            handleCancel () {
                if (this.attribute.type === 'create') {
                    this.slider.show = false
                } else {
                    this.attribute.type = 'details'
                }
            },
            handleApplayColumnsConfig (properties) {
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
            routeToHistory () {
                this.$router.push({
                    name: 'businessHistory',
                    query: {
                        from: this.$route.fullPath
                    }
                })
            },
            handleSliderBeforeClose () {
                if (this.tab.active === 'attribute' && this.attribute.type !== 'details') {
                    const $form = this.$refs.form
                    const changedValues = $form.changedValues
                    if (Object.keys(changedValues).length) {
                        return new Promise((resolve, reject) => {
                            this.$bkInfo({
                                title: this.$t('确认退出'),
                                subTitle: this.$t('退出会导致未保存信息丢失'),
                                extCls: 'bk-dialog-sub-header-center',
                                confirmFn: () => {
                                    resolve(true)
                                    this.handleCancel()
                                },
                                cancelFn: () => {
                                    resolve(false)
                                }
                            })
                        })
                    }
                    this.handleCancel()
                    return true
                }
                this.handleCancel()
                return true
            }
        }
    }
</script>

<style lang="scss" scoped>
.options-filter{
    position: relative;
    margin-right: 10px;
    .filter-selector{
        width: 115px;
        border-radius: 2px 0 0 2px;
        margin-right: -1px;
    }
    .filter-value{
        width: 320px;
        border-radius: 0 2px 2px 0;
        /deep/ .bk-form-input {
            border-radius: 0 2px 2px 0;
        }
    }
    .filter-search{
        position: absolute;
        right: 10px;
        top: 9px;
        cursor: pointer;
    }
}
.options-button{
    font-size: 0;
    .bk-button {
        width: 32px;
        padding: 0;
        /deep/ .bk-icon {
            line-height: 14px;
        }
    }
    .button-history{
        margin-right: 10px;
        border-radius: 2px 0 0 2px;
    }
    .button-setting{
        border-radius: 0 2px 2px 0;
        margin-left: -1px;
    }
}
.business-table{
    margin-top: 14px;
}
</style>

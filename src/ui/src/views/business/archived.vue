<template>
    <div class="archived-layout">
        <div class="archived-options clearfix">
            <label class="fl">{{$t('归档历史')}}</label>
            <bk-button class="fr" theme="primary" @click="back">{{$t('返回')}}</bk-button>
        </div>
        <bk-table class="archived-table"
            :pagination="pagination"
            :data="list"
            :max-height="$APP.height - 160"
            @page-change="handlePageChange"
            @page-limit-change="handleSizeChange">
            <bk-table-column v-for="column in header"
                :key="column.id"
                :prop="column.id"
                :label="column.name">
            </bk-table-column>
            <bk-table-column :label="$t('操作')" fixed="right">
                <template slot-scope="{ row }">
                    <span class="inline-block-middle"
                        v-cursor="{
                            active: !$isAuthorized(archiveAuth),
                            auth: [archiveAuth]
                        }">
                        <bk-button theme="primary" size="small"
                            :disabled="!$isAuthorized(archiveAuth)"
                            @click="handleRecovery(row)">
                            {{$t('恢复业务')}}
                        </bk-button>
                    </span>
                </template>
            </bk-table-column>
        </bk-table>
    </div>
</template>

<script>
    import { mapGetters, mapActions } from 'vuex'
    export default {
        data () {
            return {
                properties: [],
                header: [],
                list: [],
                pagination: {
                    current: 1,
                    limit: 10,
                    count: 0
                }
            }
        },
        computed: {
            ...mapGetters(['supplierAccount', 'isAdminView', 'userName']),
            ...mapGetters('userCustom', ['usercustom']),
            ...mapGetters('objectBiz', ['bizId']),
            customBusinessColumns () {
                return this.usercustom[`${this.userName}_biz_${this.isAdminView ? 'adminView' : this.bizId}_table_columns`]
            },
            archiveAuth () {
                return this.$OPERATION.BUSINESS_ARCHIVE
            }
        },
        async created () {
            try {
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
                this.setTableHeader()
                this.getTableData()
            } catch (e) {
                // ignore
            }
        },
        methods: {
            ...mapActions('objectModelProperty', ['searchObjectAttribute']),
            ...mapActions('objectBiz', ['searchBusiness', 'recoveryBusiness']),
            back () {
                this.$router.go(-1)
            },
            setTableHeader () {
                const headerProperties = this.$tools.getHeaderProperties(this.properties, this.customBusinessColumns, ['bk_biz_name'])
                this.header = [{
                    id: 'bk_biz_id',
                    name: 'ID'
                }].concat(headerProperties.map(property => {
                    return {
                        id: property['bk_property_id'],
                        name: property['bk_property_name']
                    }
                })).concat([{
                    id: 'last_time',
                    name: this.$t('更新时间')
                }])
            },
            getTableData () {
                this.searchBusiness({
                    params: this.getSearchParams(),
                    config: {
                        cancelPrevious: true,
                        requestId: 'searchArchivedBusiness'
                    }
                }).then(business => {
                    if (business.count && !business.info.length) {
                        this.pagination.current -= 1
                        this.getTableData()
                    }
                    this.pagination.count = business.count
                    this.list = this.$tools.flattenList(this.properties, business.info.map(biz => {
                        biz['last_time'] = this.$tools.formatTime(biz['last_time'], 'YYYY-MM-DD HH:mm:ss')
                        return biz
                    }))
                })
            },
            getSearchParams () {
                return {
                    condition: {
                        'bk_data_status': 'disabled'
                    },
                    fields: [],
                    page: {
                        start: (this.pagination.current - 1) * this.pagination.limit,
                        limit: this.pagination.limit,
                        sort: '-bk_biz_id'
                    }
                }
            },
            handleRecovery (biz) {
                this.$bkInfo({
                    title: this.$t('是否确认恢复业务？'),
                    subTitle: this.$t('恢复业务提示', { bizName: biz['bk_biz_name'] }),
                    confirmFn: () => {
                        this.recoveryBiz(biz)
                    }
                })
            },
            recoveryBiz (biz) {
                this.recoveryBusiness({
                    params: {
                        'bk_biz_id': biz['bk_biz_id']
                    },
                    config: {
                        cancelWhenRouteChange: false
                    }
                }).then(() => {
                    this.$http.cancel('post_searchBusiness_$ne_disabled')
                    this.$success(this.$t('恢复业务成功'))
                    this.getTableData()
                })
            },
            handleSizeChange (size) {
                this.pagination.limit = size
                this.handlePageChange(1)
            },
            handlePageChange (current) {
                this.pagination.current = current
                this.getTableData()
            }
        }
    }
</script>

<style lang="scss" scoped>
    .archived-layout{
        padding: 20px;
    }
    .archived-options{
        height: 36px;
        line-height: 36px;
        font-size: 14px;
    }
    .archived-table{
        margin-top: 20px;
    }
</style>

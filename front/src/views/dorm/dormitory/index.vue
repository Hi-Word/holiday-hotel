<template>
<div class="search">
    <add v-if="currView=='add'" @close="currView='index'" @submited="submited" />
    <edit v-if="currView=='edit'" @close="currView='index'" @submited="submited" :data="formData" />
    <Card v-show="currView=='index'">
        <Row v-show="openSearch" @keydown.enter.native="handleSearch">
            <Form ref="searchForm" :model="searchForm" inline :label-width="0">
                <Form-item ref="searchForm" :model="searchForm" inline :label-width="0" style="display:flex;">
                    <Form-item label="" prop="title">
                        <Input type="text" v-model="searchForm.title" placeholder="请输入民宿名称" clearable style="width: 200px" />
                    </Form-item>
                    <Form-item label="" prop="type">
                        <Select v-model="searchForm.type" placeholder="请选择房间类型" clearable style="width: 200px">
                            <Option value="单人间">单人间</Option>
                            <Option value="双人间">双人间</Option>
                            <Option value="套间">套间</Option>
                            <Option value="夫妻间">夫妻间</Option>
                        </Select>
                    </Form-item>
                    <Form-item label="" prop="breakfast">
                        <Select v-model="searchForm.breakfast" placeholder="请选择早餐情况" clearable style="width: 200px">
                            <Option value="不含早餐">不含早餐</Option>
                            <Option value="含单人早餐">含单人早餐</Option>
                            <Option value="含双早">含双早</Option>
                            <Option value="其他">其他</Option>
                        </Select>
                    </Form-item>
                    <Form-item style="margin-left:10px;" class="br">
                        <Button @click="handleSearch" type="primary" icon="ios-search" size="small" ghost>搜索</Button>
                        <Button @click="handleReset" type="warning" size="small" icon="md-refresh" ghost>重置</Button>
                        <Button @click="add" type="info" size="small" icon="md-add" ghost>添加</Button>
                        <Button @click="delAll" type="error" icon="md-trash" size="small" ghost>删除</Button>
                        <Button @click="excelData" type="success" icon="md-paper-plane" size="small" ghost>导出</Button>
                    </Form-item>
                    <Form-item style="position:fixed;right:50px;top:130px">
                        <Button type="info" @click="showFilterPanelFlag = !showFilterPanelFlag" class="showFilterPanelFlag" icon="md-settings" size="small" ghost>
                            列筛选</Button>
                        <Button type="warning" @click="modal1 = true" class="showFilterPanelFlag" icon="ios-help-circle-outline" size="small" ghost>
                            使用教程</Button>
                        <Modal v-model="modal1" title="使用教程">
                            <p>这里是存放模块说明的地方</p>
                        </Modal>
                    </Form-item>
                </Form-item>
            </Form>
        </Row>
        <Row class="operation" style="position:relative;">
            <transition>
                <div v-show="showFilterPanelFlag" class="filter-panel">
                    <CheckboxGroup v-model="selected">
                        <div v-for="item in mycolumns" :key="item.key">
                            <Checkbox :label="item.title" style="margin: 2px 5px"></Checkbox>
                        </div>
                    </CheckboxGroup>
                </div>
            </transition>
        </Row>
        <Row v-show="openTip"> </Row>
        <Row>
            <Table :loading="loading" :height="tableHeight" border stripe size="small" :columns="columns" :data="data" ref="table" sortable="custom" @on-row-dblclick="dblclickRow" @on-sort-change="changeSort" @on-selection-change="changeSelect" @on-row-click="rowClick" :row-class-name="rowClassNmae"></Table>
        </Row>
        <Row type="flex" justify="end" class="page">
            <Page :current="searchForm.pageNumber" :total="total" :page-size="searchForm.pageSize" @on-change="changePage" @on-page-size-change="changePageSize" :page-size-opts="[15,20,50]" size="small" show-total show-elevator show-sizer></Page>
        </Row>
    </Card>
    <Modal v-model="disModal" title="查询民宿评论" footer-hide width="80">
        <Row>
            <Table :loading="disLoading" :height="tableHeight" border stripe size="small" :columns="disColumns" :data="disList"></Table>
        </Row>
    </Modal>
    <Modal v-model="addDisModal" title="新增民宿评论" ok-text="确认评论" width="600" @on-ok="addDisFx">
        <Input v-model="myAddDisText" size="large" maxlength="500" :rows="10" show-word-limit type="textarea" placeholder="请输入您的评论..." style="width:100%" />
    </Modal>
    <Modal v-model="addOrderModal" title="新增预约" ok-text="确认预约" width="600" @on-ok="addOrderFx">
        <DatePicker v-model="myAddDisDate2" type="date" placeholder="请选择预约的日期" format="yyyy-MM-dd" style="width: 100%" @on-change="changeAddDisDate"></DatePicker>
    </Modal>
</div>
</template>

<script>
import {
    getDormitoryList,
    deleteDormitory,
    getAllDiscuss,
    addMyDiscuss,
    addMyOrder
} from "./api.js";
import add from "./add.vue";
import edit from "./edit.vue";
export default {
    name: "single-window",
    components: {
        add,
        edit
    },
    data() {
        return {
            addOrderModal: false,
            myAddDisDate: "",
            myAddDisDate2: "",
            addDisModal: false,
            disLoading: false,
            disModal: false,
            tableHeight: 0,
            selected: [
                "选择",
                "序号",
                "民宿名称",
                "面积",
                "房间类型",
                "早餐情况",
                "价格",
                "备注",
                "创建时间",
                "创建者",
                "修改时间",
                "修改者",
                "操作",
            ],
            modal1: false,
            openSearch: true,
            openTip: true,
            formData: {},
            currView: "index",
            loading: true,
            searchForm: {
                pageNumber: 1,
                pageSize: 15,
                sort: "createTime",
                order: "desc",
            },
            selectList: [],
            selectCount: 0,
            selectRow: 0,
            disColumns: [{
                    title: "评论人",
                    key: "userName",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "评论内容",
                    key: "content",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "评论时间",
                    key: "discussTime",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
            ],
            columns: [{
                    type: "selection",
                    width: 60,
                    title: "选择",
                    align: "center",
                    fixed: "left",
                },
                {
                    title: "序号",
                    width: 85,
                    align: "center",
                    fixed: "left",
                    sortType: true,
                    render: (h, params) => {
                        return h(
                            "span",
                            params.index +
                            (this.searchForm.pageNumber - 1) * this.searchForm.pageSize +
                            1
                        );
                    },
                },
                {
                    title: "民宿名称",
                    key: "title",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "面积",
                    key: "areaSize",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "房间类型",
                    key: "type",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "早餐情况",
                    key: "breakfast",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "价格",
                    key: "price",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "备注",
                    key: "remark",
                    minWidth: 120,
                    tooltip: true,
                    sortable: false,
                },
                {
                    title: "创建时间",
                    key: "createTime",
                    sortable: true,
                    sortType: "desc",
                    minWidth: 180,
                    align: "center",
                    tooltip: true,
                },
                {
                    title: "创建者",
                    key: "createBy",
                    sortable: true,
                    sortType: "desc",
                    minWidth: 100,
                    align: "center",
                    tooltip: true,
                },
                {
                    title: "修改时间",
                    key: "updateTime",
                    minWidth: 180,
                    align: "center",
                    tooltip: true,
                },
                {
                    title: "修改者",
                    key: "updateBy",
                    minWidth: 100,
                    align: "center",
                    tooltip: true,
                },
                {
                    title: "操作",
                    key: "action",
                    align: "center",
                    width: 360,
                    fixed: "right",
                    render: (h, params) => {
                        return h("div", [
                            h(
                                "Button", {
                                    props: {
                                        type: "info",
                                        size: "small",
                                        icon: "ios-add",
                                        ghost: true
                                    },
                                    style: {
                                        marginRight: "5px"
                                    },
                                    on: {
                                        click: () => {
                                            this.orderFx(params.row);
                                        }
                                    }
                                },
                                "预约"
                            ),
                            h(
                                "Button", {
                                    props: {
                                        type: "primary",
                                        size: "small",
                                        icon: "ios-create-outline",
                                        ghost: true
                                    },
                                    style: {
                                        marginRight: "5px"
                                    },
                                    on: {
                                        click: () => {
                                            this.edit(params.row);
                                        }
                                    }
                                },
                                "编辑"
                            ),
                            h(
                                "Button", {
                                    props: {
                                        type: "success",
                                        size: "small",
                                        icon: "ios-copy",
                                        ghost: true
                                    },
                                    style: {
                                        marginRight: "5px"
                                    },
                                    on: {
                                        click: () => {
                                            this.disFx(params.row);
                                        }
                                    }
                                },
                                "评论"
                            ),
                            h(
                                "Button", {
                                    props: {
                                        type: "error",
                                        size: "small",
                                        icon: "md-trash",
                                        ghost: true
                                    },
                                    on: {
                                        click: () => {
                                            this.remove(params.row);
                                        }
                                    }
                                },
                                "删除"
                            )
                        ]);
                    }
                }
            ],
            data: [],
            pageNumber: 1,
            pageSize: 10,
            total: 0,
            showFilterPanelFlag: false,
            disList: [],
            myAddDisText: ""
        };
    },
    methods: {
        init() {
            this.getDataList();
        },
        changeAddDisDate(e) {
            this.myAddDisDate = e;
        },
        orderFx(e) {
            this.mySelectRow = e;
            var date = new Date();
            this.myAddDisDate = date.getFullYear() + "-" + (date.getMonth() + 1 < 10 ? "0" + (date.getMonth() + 1) : (date.getMonth() + 1)) + "-" + (date.getDate() < 10 ? "0" + date.getDate() : date.getDate());
            this.myAddDisDate2 = date.getFullYear() + "-" + (date.getMonth() + 1 < 10 ? "0" + (date.getMonth() + 1) : (date.getMonth() + 1)) + "-" + (date.getDate() < 10 ? "0" + date.getDate() : date.getDate());
            this.addOrderModal = true;
        },
        addOrderFx() {
            addMyOrder({
                id: this.mySelectRow.id,
                date: this.myAddDisDate
            }).then(res => {
                if (res.success) {
                    this.$Message.success("预约成功");
                }
            })
        },
        addDisFx() {
            addMyDiscuss({
                id: this.mySelectRow.id,
                reply: "",
                text: this.myAddDisText
            }).then(res => {
                if (res.success) {
                    this.$Message.success("评论成功");
                }
            })
        },
        disFx(e) {
            this.addDisModal = true;
            this.mySelectRow = e;
        },
        dblclickRow(e) {
            this.mySelectRow = e;
            this.getAllDiscussFx();
            this.disModal = true;
        },
        getAllDiscussFx() {
            var that = this;
            that.disLoading = true;
            var f = {};
            f.dormitoryId = this.mySelectRow.id;
            getAllDiscuss(f).then(res => {
                if (res.success) {
                    that.disLoading = false;
                    that.disList = res.result;
                }
            })
        },
        submited() {
            this.currView = "index";
            this.getDataList();
        },
        changePage(v) {
            this.searchForm.pageNumber = v;
            this.getDataList();
            this.clearSelectAll();
        },
        changePageSize(v) {
            this.searchForm.pageSize = v;
            this.getDataList();
        },
        rowClick(row, index) {
            this.selectRow = row;
        },
        rowClassNmae(row, index) {
            if (row.id == this.selectRow.id) {
                return "rowClassNmaeColor";
            }
            return "";
        },
        excelData() {
            this.$refs.table.exportCsv({
                filename: "导出结果",
            });
        },
        handleSearch() {
            this.searchForm.pageNumber = 1;
            this.searchForm.pageSize = 15;
            this.getDataList();
        },
        handleReset() {
            this.$refs.searchForm.resetFields();
            this.searchForm.pageNumber = 1;
            this.searchForm.pageSize = 15;
            this.getDataList();
        },
        changeSort(e) {
            this.searchForm.sort = e.key;
            this.searchForm.order = e.order;
            if (e.order === "normal") {
                this.searchForm.order = "";
            }
            this.getDataList();
        },
        clearSelectAll() {
            this.$refs.table.selectAll(false);
        },
        changeSelect(e) {
            this.selectList = e;
            this.selectCount = e.length;
        },
        getDataList() {
            this.loading = true;
            getDormitoryList(this.searchForm).then(res => {
                this.loading = false;
                if (res.success) {
                    this.data = res.result.records;
                    this.total = res.result.total;
                }
            });
        },
        add() {
            this.currView = "add";
        },
        edit(v) {
            for (let attr in v) {
                if (v[attr] == null) {
                    v[attr] = "";
                }
            }
            let str = JSON.stringify(v);
            let data = JSON.parse(str);
            this.formData = data;
            this.currView = "edit";
        },
        remove(v) {
            this.$Modal.confirm({
                title: "确认删除",
                content: "您确认要删除?",
                loading: true,
                onOk: () => {
                    deleteDormitory({
                        ids: v.id
                    }).then(res => {
                        this.$Modal.remove();
                        if (res.success) {
                            this.$Message.success("操作成功");
                            this.getDataList();
                        }
                    });
                }
            });
        },
        delAll() {
            if (this.selectCount <= 0) {
                this.$Message.warning("您还未选择要删除的数据");
                return;
            }
            this.$Modal.confirm({
                title: "确认删除",
                content: "您确认要删除所选的数据?",
                loading: true,
                onOk: () => {
                    let ids = "";
                    this.selectList.forEach(function (e) {
                        ids += e.id + ",";
                    });
                    ids = ids.substring(0, ids.length - 1);
                    deleteDormitory({
                        ids: ids
                    }).then(res => {
                        this.$Modal.remove();
                        if (res.success) {
                            this.$Message.success("操作成功");
                            this.clearSelectAll();
                            this.getDataList();
                        }
                    });
                }
            });
        }
    },
    mounted() {
        this.init();
        this.tableHeight = Number(window.innerHeight - 290);
        this.mycolumns = this.columns;
        let showcolumns = [];
        for (var i = 0; i < this.selected.length; i++) {
            var item = this.selected[i];
            for (var j = 0; j < this.columns.length; j++) {
                if (this.columns[j].title == item) {
                    showcolumns.push(this.columns[j]);
                }
            }
        }
        this.columns = showcolumns;
    },
    watch: {
        selected: function (newcolumns) {
            let showcolumns = [];
            for (var i = 0; i < this.mycolumns.length; i++) {
                var item = this.mycolumns[i];
                if (item.title == undefined) showcolumns.push(item);
                else if (newcolumns.contains(item.title)) showcolumns.push(item);
            }
            this.columns = showcolumns;
        },
    },
};
</script>

<style lang="less">
.search {
    .operation {
        margin-bottom: 2vh;
    }

    .select-count {
        font-weight: 600;
        color: #40a9ff;
    }

    .select-clear {
        margin-left: 10px;
    }

    .page {
        margin-top: 2vh;
    }

    .drop-down {
        margin-left: 5px;
    }
}

.filter-panel {
    width: 166px;
    min-height: 120px;
    height: 200px;
    position: absolute;
    background-color: white;
    z-index: 9999;
    margin-left: 1px;
    overflow-y: scroll;
    border: 1px solid blue;
    top: 35px;
    right: 10px;
}

.openSearch {
    position: absolute;
    right: 240px;
}

.openTip {
    position: absolute;
    right: 130px;
}

.showFilterPanelFlag {
    position: static !important;
    right: 10px;
    margin-right: 10px;
}

.ivu-table td {
    height: 38px !important;
}

.ivu-table-cell-with-expand {
    height: 38px !important;
    line-height: 38px !important;
}

.ivu-table .rowClassNmaeColor td {
    background-color: #b0b3b6 !important;
    color: #ffffff !important;
    font-size: 12px;
}
</style>

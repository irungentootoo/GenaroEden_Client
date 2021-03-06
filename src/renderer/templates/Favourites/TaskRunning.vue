<style scoped>
.right-container {
  padding: 15px 15px 0 15px;
  box-sizing: border-box;
}
.top-bar {
  padding: 0 0 10px;
}
.el-tab-pane {
  display: flex;
  flex-grow: 1;
}
.task-type {
  width: 160px;
  display: inline-block;
}
.action-cell,
.action-cell span {
  display: flex;
  align-items: center;
}
.process-bar i {
  vertical-align: bottom;
}

.row-action {
  visibility: hidden;
}
.file-row:hover .row-action {
  visibility: visible;
}

.not-actived {
  color: #ccc;
}
</style>

<template>
    <div class="fullheight right-container">
        <div class="top-bar">
            {{ $t("dashboard.recent.runningtask") }}
        </div>
        <el-table :data="taskListNotSuccess" class="files-table" row-class-name="file-row">
            <el-table-column prop="filename" :label="$t('dashboard.recent.filename')" :show-overflow-tooltip="true" min-width="200">
                <template slot-scope="scope">
                    <font-awesome-icon :icon="file2Icon(scope.row.filePath).icon" v-bind:style="{ color: file2Icon(scope.row.filePath).color }" />
                    <span style="margin-left: 10px">{{ getFileName(scope.row.filePath) }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="totalBytes" :label="$t('dashboard.recent.size')" min-width="200">
                <template slot-scope="scope">
                    <div class="process-bar">
                        <el-progress :status="getStatusStr(scope.row.taskState)" :text-inside="true" :stroke-width="15" :percentage="Number.parseInt(scope.row.progress * 100)"></el-progress>
                        <div>
                            <i v-if="scope.row.taskType === TASK_TYPE.DOWNLOAD" :class="['material-icons', {[`not-actived`]: scope.row.taskState !== TASK_STATE.INPROGRESS}]">file_download</i>
                            <i v-if="scope.row.taskType === TASK_TYPE.UPLOAD" :class="['material-icons', {[`not-actived`]: scope.row.taskState !== TASK_STATE.INPROGRESS}]">file_upload</i>
                            <span v-if="scope.row.taskState === TASK_STATE.INPROGRESS">{{humanSize(scope.row.totalBytes * scope.row.progress)}}/{{humanSize(scope.row.totalBytes)}}</span>
                            <span v-else-if="scope.row.taskState === TASK_STATE.INIT">waiting...</span>
                            <span v-else-if="scope.row.taskState === TASK_STATE.CANCEL">canceled</span>
                            <span v-else-if="scope.row.taskState === TASK_STATE.ERROR">error: {{ scope.row.errorMessage }}</span>
                        </div>
                    </div>
                </template>
            </el-table-column>
            <el-table-column prop="created" :label="$t('dashboard.recent.created')" width="180" :formatter="formatTime"></el-table-column>
            <el-table-column prop="folderName" :label="$t('dashboard.recent.folder')" width="100">
                <template slot-scope="scope">
                    <router-link :to="`/folder/${scope.row.bucketId}`">{{ scope.row.folderName }}</router-link>
                </template>
            </el-table-column>
            <el-table-column width="100" label="">
                <template slot-scope="scope">
                    <div class="action-cell">
                        <el-button v-if="scope.row.taskState === TASK_STATE.INPROGRESS" class="row-action" @click="cancelTask(scope.row)" type="text" size="small">
                            <i class="material-icons">cancel</i>
                        </el-button>
                        <el-tooltip v-else placement="top" :manual="true" :value="scope.row.taskId === deleteTaskId">
                            <div slot="content">
                                <p>Are you sure to delete?</p>
                                <div style="text-align: right; margin: 0">
                                    <el-button size="mini" type="text" @click="deleteTaskId = null">{{ $t('el.messagebox.cancel') }}</el-button>
                                    <el-button type="primary" size="mini" @click="removeTask(scope.row)">{{ $t('el.messagebox.confirm') }}</el-button>
                                </div>
                            </div>
                            <el-button class="row-action" @click="deleteTaskId = scope.row.taskId" type="text" size="small">
                                <i class="material-icons">close</i>
                            </el-button>
                        </el-tooltip>
                    </div>
                </template>
            </el-table-column>
            <span slot="empty">{{ $t('dashboard.recent.tip1') }}</span>
        </el-table>
    </div>
</template>

<script>

import { fileName2Icon } from "../../utils/file2icon";
import FontAwesomeIcon from '@fortawesome/vue-fontawesome'
import { TASK_STATE, TASK_TYPE } from "../../../config"
const moment = require('moment');
const humanSize = require('human-size');
export default {
    data() {
        return {
            TASK_STATE,
            TASK_TYPE,
            deleteTaskId: null,
            fileTableColums: [{
                title: this.$t('dashboard.recent.filename'),
                key: 'filename',
                width: 230
            }, {
                title: this.$t('dashboard.recent.filepath'),
                key: 'filePath',
                width: 230
            }, {
                title: this.$t('dashboard.myfiles.foldername'),
                key: 'bucketName',
                width: 230
            }]
        }
    },
    computed: {
        taskList() {
            return this.$store.state.TaskList.tasks;
        },
        taskListSuccess() {
            return this.$store.state.TaskList.tasks.filter(task => task.taskState === TASK_STATE.SUCCESS);
        },
        taskListNotSuccess() {
            return this.$store.state.TaskList.tasks.filter(task => task.taskState !== TASK_STATE.SUCCESS);
        }
    },
    methods: {
        formatTime(row, column) {
            return moment(row.created).format("MM/DD/YYYY hh:mm a")
        },
        formatSize(row, column) {
            return humanSize(row.totalBytes)
        },
        file2Icon(name) {
            return fileName2Icon(name)
        },
        getFileName(filePath) {
            return filePath.split('/').pop()
        },
        humanSize(bytes) {
            return humanSize(bytes)
        },
        getStatusStr(status) {
            switch (status) {
                case TASK_STATE.SUCCESS:
                    return 'success'
                case TASK_STATE.ERROR:
                    return 'exception'
                default:
                    return null
            }
        },
        ShowUploadItemInFolder(item) {
            if (item.taskType === TASK_TYPE.DOWNLOAD) {
                const { shell } = require('electron')
                const fs = require('fs');
                if (fs.existsSync(item.filePath)) {
                    shell.showItemInFolder(item.filePath)
                } else {
                    this.$message(this.$t('dashboard.recent.filenotexist'));
                }
            } else if (item.taskType === TASK_TYPE.UPLOAD) {
                let bucketList = this.$store.state.BucketList.buckets;
                let bucketExist = false;
                bucketList.every(bucket => {
                    if (bucket.name !== item.folderName) return true;
                    bucketExist = true;
                    return false;
                });
                if (bucketExist) this.$router.push({ path: '/folder/' + item.bucketId, query: { folderName: item.folderName } });
                else this.$message.error(this.$t('dashboard.recent.foldernotexist', {folderName: item.folderName}));
            }
        },
        cancelTask(item) {
            this.$store.dispatch('taskListCancel', { taskId: item.taskId });
        },
        removeTask(item) {
            this.$store.commit('taskListRemove', { taskId: item.taskId })
        }
    },
    components: {
        FontAwesomeIcon
    }
}
</script>
<template>
    <el-dialog v-model="visibleInner" title="修改评分与评价" width="500px">
        <div v-loading="loading" element-loading-text="加载中...">
            <el-form label-width="80px">
                <el-form-item label="评分">
                    <el-rate v-model="rate" :max="10" />
                </el-form-item>
                <el-form-item label="评语">
                    <el-input v-model="comment" type="textarea" :rows="4" maxlength="200" show-word-limit />
                </el-form-item>
            </el-form>
            <div style="display:flex;justify-content:flex-end;gap:8px;margin-top:12px">
                <el-button @click="visibleInner = false">取消</el-button>
                <el-button type="primary" :loading="saving" @click="save">保存</el-button>
            </div>
        </div>
    </el-dialog>
</template>

<script setup lang="ts">
import { ref, watch, inject } from 'vue'
import { ElNotification } from 'element-plus'
import api from '../api/client'

const props = defineProps<{ subjectId: number, modelValue: boolean }>()
const emit = defineEmits<{
    (e: 'update:modelValue', v: boolean): void
    (e: 'saved', payload: { rate: number, comment: string }): void
}>()

const appState = inject('appState') as any
const visibleInner = ref(false)
const loading = ref(false)
const saving = ref(false)
const rate = ref<number>(0)
const comment = ref('')

watch(() => props.modelValue, (v) => {
    visibleInner.value = v
    if (v) load()
})

watch(visibleInner, (v) => emit('update:modelValue', v))

const load = async () => {
    if (!appState?.username) return
    loading.value = true
    try {
        const { data } = await api.get(`/v0/users/${appState.username}/collections/${props.subjectId}`, { useToken: true })
        rate.value = data?.rate ?? 0
        comment.value = data?.comment ?? ''
    } catch (error: any) {
        ElNotification({ title: '错误', message: error?.response?.data?.message || '加载失败', type: 'error' })
    } finally {
        loading.value = false
    }
}

const save = async () => {
    saving.value = true
    try {
        if (!rate.value || rate.value < 1) {
            ElNotification({ title: '提示', message: '请选择 1-10 的整数评分', type: 'warning' })
            saving.value = false
            return
        }
        await api.post(`/v0/users/-/collections/${props.subjectId}`, { rate: rate.value, comment: comment.value }, { useToken: true })
        ElNotification({ title: '成功', message: '已保存评分与评价', type: 'success' })
        emit('saved', { rate: rate.value, comment: comment.value })
        visibleInner.value = false
    } catch (error: any) {
        ElNotification({ title: '错误', message: error?.response?.data?.message || '保存失败', type: 'error' })
    } finally {
        saving.value = false
    }
}
</script>

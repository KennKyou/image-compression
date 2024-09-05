<script setup>
import { ref, computed } from 'vue';
import { ElMessage } from "element-plus";
import imageCompression from 'browser-image-compression';
import { UploadFilled } from '@element-plus/icons-vue'

const files = ref([]);
const isCompressing = ref(false);
const previewVisible = ref(false)
const previewIndex = ref(0)

const handleFileChange = (uploadFile, uploadFiles) => {
  files.value = uploadFiles.map(file => ({
    raw: file.raw,
    url: URL.createObjectURL(file.raw),
    compressed: null,
    compressedUrl: null,
  }));
}

const clearFiles = () => {
  files.value = [];
}

const showPreview = (index) => {
  previewIndex.value = index
  previewVisible.value = true
}

const closePreview = () => {
  previewVisible.value = false
}

const compressImages = async () => {
  if (files.value.length === 0) {
    ElMessage.warning('請先選擇圖片');
    return;
  }

  isCompressing.value = true;
  const options = {
    maxSizeMB: 1,
    maxWidthOrHeight: 1920,
    useWebWorker: true
  }

  try {
    for (let file of files.value) {
      const compressedBlob = await imageCompression(file.raw, options);
      const compressedFile = new File([compressedBlob], file.raw.name, { type: compressedBlob.type });
      file.compressed = compressedFile;
      file.compressedUrl = URL.createObjectURL(compressedFile);
    }
    ElMessage.success('圖片壓縮成功');
  } catch (error) {
    console.error('圖片壓縮失敗', error);
    ElMessage.error('圖片壓縮失敗');
  } finally {
    isCompressing.value = false;
  }
}

const downloadCompressedImages = () => {
  const compressedFiles = files.value.filter(file => file.compressed);
  if (compressedFiles.length === 0) {
    ElMessage.warning('沒有可下載的壓縮圖片');
    return;
  }

  compressedFiles.forEach(file => {
    const link = document.createElement('a');
    link.href = file.compressedUrl;
    link.download = file.compressed.name;
    link.click();
  });
}

const getFileSize = (size) => {
  return (size / 1024 / 1024).toFixed(2) + ' MB';
}

const getCompressionRatio = (original, compressed) => {
  if (original && compressed) {
    return ((1 - compressed / original) * 100).toFixed(2) + '%';
  }
  return '尚未壓縮';
}

const totalFileSize = computed(() => {
  return files.value.reduce((total, file) => total + file.raw.size, 0);
});

const totalCompressedFileSize = computed(() => {
  return files.value.reduce((total, file) => total + (file.compressed ? file.compressed.size : 0), 0);
});

const compressionRatio = computed(() => {
  if (totalFileSize.value > 0 && totalCompressedFileSize.value > 0) {
    return ((1 - totalCompressedFileSize.value / totalFileSize.value) * 100).toFixed(2) + '%';
  }
  return '尚未壓縮';
});
</script>

<template>
  <div class="image-compression">

    <el-upload
      class="upload-area"
      drag
      action="#"
      :auto-upload="false"
      :on-change="handleFileChange"
      accept="image/*"
      multiple
      :file-list="files"
      :show-file-list="false"
    >
      <el-icon class="el-icon--upload"><upload-filled /></el-icon>
      <div class="el-upload__text">
        拖曳圖片到此處或 <em>點擊上傳</em>
      </div>
    </el-upload>
    
    <div v-if="files.length > 0" class="preview-container">
      <el-image
        v-for="(file, index) in files"
        :key="index"
        :src="file.compressedUrl || file.url"
        fit="cover"
        class="preview-image"
        @click="showPreview(index)"
      />
    </div>

    <el-image-viewer
      v-if="previewVisible"
      :url-list="files.map(f => f.compressedUrl || f.url)"
      :initial-index="previewIndex"
      @close="closePreview"
    />
    
    <div v-if="files.length > 0" class="file-info">
      <div class="table-wrapper">
        <el-table :data="files" style="width: 100%" class="desktop-table" table-layout="auto">
          <el-table-column prop="raw.name" label="文件名稱" min-width="120"></el-table-column>
          <el-table-column label="原始大小" min-width="100">
            <template #default="scope">
              {{ getFileSize(scope.row.raw.size) }}
            </template>
          </el-table-column>
          <el-table-column label="壓縮後大小" min-width="100">
            <template #default="scope">
              {{ scope.row.compressed ? getFileSize(scope.row.compressed.size) : '尚未壓縮' }}
            </template>
          </el-table-column>
          <el-table-column label="壓縮比率" min-width="100">
            <template #default="scope">
              {{ getCompressionRatio(scope.row.raw.size, scope.row.compressed?.size) }}
            </template>
          </el-table-column>
        </el-table>
      </div>

      <div class="mobile-cards">
        <el-card v-for="(file, index) in files" :key="index" class="file-card">
          <h3>{{ file.raw.name }}</h3>
          <p>原始大小：{{ getFileSize(file.raw.size) }}</p>
          <p>壓縮後大小：{{ file.compressed ? getFileSize(file.compressed.size) : '尚未壓縮' }}</p>
          <p>壓縮比率：{{ getCompressionRatio(file.raw.size, file.compressed?.size) }}</p>
        </el-card>
      </div>

      <el-card class="total-info">
        <el-descriptions :column="1" border>
          <el-descriptions-item label="總原始大小">{{ getFileSize(totalFileSize) }}</el-descriptions-item>
          <el-descriptions-item label="總壓縮後大小">{{ getFileSize(totalCompressedFileSize) }}</el-descriptions-item>
          <el-descriptions-item label="總壓縮比率">{{ compressionRatio }}</el-descriptions-item>
        </el-descriptions>
      </el-card>
    </div>
    
    <div class="action-buttons">
      <el-button type="primary" @click="compressImages" :loading="isCompressing" :disabled="files.length === 0" class="compress-button">
        {{ isCompressing ? '壓縮中...' : '壓縮圖片' }}
      </el-button>
      <el-button type="success" @click="downloadCompressedImages" :disabled="!files.some(file => file.compressed)">
        下載壓縮後的圖片
      </el-button>
      <el-button type="warning" @click="clearFiles" :disabled="files.length === 0">
        清除所有文件
      </el-button>
    </div>
  </div>
</template>

<style scoped>
.image-compression {
  padding: 20px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
}

.page-title {
  margin-bottom: 20px;
  color: #409EFF;
  font-size: 1.5rem;
}

.upload-area {
  margin-bottom: 20px;
  border-radius: 6px;
  transition: border-color 0.3s;
}

.upload-area:hover {
  border-color: #409EFF;
}

.preview-container {
  margin-top: 20px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(100px, 1fr));
  gap: 10px;
  justify-content: center;
}

.preview-image {
  width: 100%;
  height: 100px;
  object-fit: cover;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s;
  cursor: pointer;
}

.preview-image:hover {
  transform: scale(1.05);
}

.file-info {
  margin-top: 20px;
  width: 100%;
}

.table-wrapper {
  width: 100%;
  overflow-x: auto;
}

.el-table {
  margin-top: 20px;
  width: 100%;
  min-width: 600px;
}

.el-table :deep(.el-table__inner-wrapper) {
  width: 100%;
}

.el-table :deep(.el-table__body),
.el-table :deep(.el-table__header) {
  width: 100% !important;
}

.el-table :deep(.el-table__cell) {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.total-info {
  margin-top: 20px;
}

.action-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  margin-top: 20px;
}

.compress-button {
  flex-grow: 1;
  min-width: 150px;
}

.el-table :deep(.el-table__row) {
  transition: background-color 0.3s;
}

.el-table :deep(.el-table__row:hover) {
  background-color: #f5f7fa;
}

.el-button {
  transition: transform 0.2s, box-shadow 0.2s;
}

.el-button:hover {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.el-button:active {
  transform: translateY(0);
  box-shadow: none;
}

.desktop-table {
  display: table;
}

.mobile-cards {
  display: none;
}

.file-card {
  margin-bottom: 10px;
}

@media (max-width: 768px) {
  .image-compression {
    padding: 10px;
  }

  .page-title {
    font-size: 1.2rem;
  }

  .preview-container {
    grid-template-columns: repeat(auto-fill, minmax(80px, 1fr));
  }

  .preview-image {
    height: 80px;
  }

  .action-buttons {
    flex-direction: column;
  }

  .compress-button,
  .action-buttons .el-button {
    width: 100%;
  }

  .desktop-table {
    display: none;
  }

  .mobile-cards {
    display: block;
  }

  .file-card {
    margin-bottom: 15px;
  }

  .file-card h3 {
    font-size: 16px;
    margin-bottom: 10px;
  }

  .file-card p {
    font-size: 14px;
    margin: 5px 0;
  }

  .el-table {
    font-size: 12px;
  }

  .el-table :deep(.el-table__cell) {
    padding: 5px !important;
  }

  .el-button, .el-button+.el-button {
    margin-left: 0;
  }
}

@media (max-width: 480px) {
  .preview-container {
    grid-template-columns: repeat(auto-fill, minmax(60px, 1fr));
  }

  .preview-image {
    height: 60px;
  }
}
</style>
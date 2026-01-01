<template>
    <v-card variant="outlined" class="mt-2">
        <v-card-text>
            <div class="d-flex align-center ga-2">
                <!-- Record Button -->
                <v-btn
                    v-if="!isRecording && !audioBlob"
                    color="error"
                    @click="startRecording"
                    :disabled="!mediaRecorderSupported"
                >
                    <v-icon>mdi-microphone</v-icon>
                    {{ $t('Record') }}
                </v-btn>

                <!-- Stop Button -->
                <v-btn
                    v-if="isRecording"
                    color="warning"
                    @click="stopRecording"
                >
                    <v-icon>mdi-stop</v-icon>
                    {{ $t('Stop') }}
                </v-btn>

                <!-- Upload Button -->
                <v-btn
                    v-if="audioBlob && !isUploading"
                    color="primary"
                    @click="uploadAudio"
                >
                    <v-icon>mdi-upload</v-icon>
                    {{ $t('Upload') }}
                </v-btn>

                <!-- Status Text -->
                <span v-if="isRecording" class="text-error">
                    <v-icon size="small" class="mr-1">mdi-record</v-icon>
                    {{ $t('Recording') }}... {{ recordingTime }}s
                </span>

                <span v-if="isUploading">
                    <v-progress-circular indeterminate size="20" width="2" class="mr-2"></v-progress-circular>
                    {{ $t('Uploading') }}...
                </span>

                <span v-if="!mediaRecorderSupported" class="text-warning">
                    {{ $t('RecordingNotSupported') }}
                </span>

                <!-- Delete Button (if file exists) -->
                <v-btn
                    v-if="props.existingFile || audioBlob"
                    icon
                    size="small"
                    variant="text"
                    @click="deleteVoiceFile"
                >
                    <v-icon>mdi-delete</v-icon>
                </v-btn>
            </div>

            <!-- Existing File Display -->
            <div v-if="props.existingFile && !audioBlob" class="mt-2">
                <v-chip color="success" size="small">
                    <v-icon start>mdi-check</v-icon>
                    {{ props.existingFile.name }}
                </v-chip>
            </div>

            <!-- Error Display -->
            <v-alert v-if="error" type="error" density="compact" class="mt-2">
                {{ error }}
            </v-alert>
        </v-card-text>
    </v-card>
</template>

<script setup lang="ts">
import { ref, onUnmounted } from 'vue';
import type { UserFileView } from '@/openapi';
import { UserFileViewFromJSON } from '@/openapi';
import { getCookie } from '@/utils/cookie';

interface Props {
    existingFile?: UserFileView | null;
}

const props = withDefaults(defineProps<Props>(), {
    existingFile: null
});

interface Emits {
    (e: 'file-uploaded', file: UserFileView): void;
    (e: 'file-deleted'): void;
}

const emit = defineEmits<Emits>();

// State
const isRecording = ref(false);
const isUploading = ref(false);
const audioBlob = ref<Blob | null>(null);
const mediaRecorder = ref<MediaRecorder | null>(null);
const recordingTime = ref(0);
const recordingInterval = ref<number | null>(null);
const error = ref<string | null>(null);

// Check MediaRecorder support
const mediaRecorderSupported = ref(typeof MediaRecorder !== 'undefined');

// Start recording
async function startRecording() {
    error.value = null;

    try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });

        // Determine supported MIME type
        let mimeType = 'audio/webm;codecs=opus';
        if (!MediaRecorder.isTypeSupported(mimeType)) {
            mimeType = 'audio/webm';
        }
        if (!MediaRecorder.isTypeSupported(mimeType)) {
            mimeType = 'audio/mp4';
        }

        mediaRecorder.value = new MediaRecorder(stream, { mimeType });
        const chunks: Blob[] = [];

        mediaRecorder.value.ondataavailable = (event) => {
            if (event.data.size > 0) {
                chunks.push(event.data);
            }
        };

        mediaRecorder.value.onstop = () => {
            audioBlob.value = new Blob(chunks, { type: mimeType });
            stream.getTracks().forEach(track => track.stop());

            if (recordingInterval.value) {
                clearInterval(recordingInterval.value);
                recordingInterval.value = null;
            }
        };

        mediaRecorder.value.start();
        isRecording.value = true;
        recordingTime.value = 0;

        // Update recording time
        recordingInterval.value = window.setInterval(() => {
            recordingTime.value++;
        }, 1000);

    } catch (err: any) {
        error.value = err.message || 'Failed to access microphone';
        console.error('Recording error:', err);
    }
}

// Stop recording
function stopRecording() {
    if (mediaRecorder.value && isRecording.value) {
        mediaRecorder.value.stop();
        isRecording.value = false;
    }
}

// Upload audio
async function uploadAudio() {
    if (!audioBlob.value) return;

    error.value = null;
    isUploading.value = true;

    try {
        // Get file extension from blob type
        const mimeType = audioBlob.value.type;
        let extension = '.webm';
        if (mimeType.includes('mp4')) extension = '.m4a';
        if (mimeType.includes('ogg')) extension = '.ogg';

        // Create FormData
        const formData = new FormData();
        const fileName = `voice_${Date.now()}${extension}`;
        formData.append('name', fileName);
        formData.append('file', audioBlob.value, fileName);

        // Upload using fetch (like UserFileField does)
        const response = await fetch('/api/user-file/', {
            method: 'POST',
            headers: { 'X-CSRFToken': getCookie('csrftoken') },
            body: formData
        });

        if (!response.ok) {
            throw new Error('Upload failed');
        }

        const data = await response.json();
        const uploadedFile = UserFileViewFromJSON(data);

        // Emit success
        emit('file-uploaded', uploadedFile);
        audioBlob.value = null;
        recordingTime.value = 0;

    } catch (err: any) {
        error.value = err.message || 'Failed to upload audio';
        console.error('Upload error:', err);
    } finally {
        isUploading.value = false;
    }
}

// Delete voice file
function deleteVoiceFile() {
    audioBlob.value = null;
    recordingTime.value = 0;
    emit('file-deleted');
}

// Cleanup on unmount
onUnmounted(() => {
    if (recordingInterval.value) {
        clearInterval(recordingInterval.value);
    }
    if (mediaRecorder.value && isRecording.value) {
        mediaRecorder.value.stop();
    }
});
</script>

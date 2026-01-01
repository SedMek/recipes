# Work Log: Voice Message Recording for Recipe Steps

**Project:** Tandoor Recipe Manager
**Feature:** Add voice message recording capability to recipe steps
**Date Started:** 2026-01-01
**Status:** Planning Phase

---

## Table of Contents
1. [Feature Overview](#feature-overview)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Implementation Plan](#implementation-plan)
4. [Technical Considerations](#technical-considerations)
5. [Testing Strategy](#testing-strategy)
6. [Future Enhancements](#future-enhancements)

---

## Feature Overview

### Goal
Allow users to record and attach voice messages to recipe steps alongside written text instructions. This enables:
- Hands-free recipe following while cooking
- Additional context and tips in the creator's voice
- Accessibility for users who prefer audio instructions
- Multi-modal recipe content (text + voice)

### User Stories
1. As a recipe creator, I want to record voice notes for each step so I can provide verbal tips and context
2. As a recipe follower, I want to play voice messages while cooking so I can keep my hands free
3. As a recipe editor, I want to manage (record, delete, re-record) voice messages per step
4. As a recipe viewer, I want to see visual indicators when a step has an associated voice message

---

## Current Architecture Analysis

### Technology Stack
**Backend:**
- Django 5.2.9 + Django REST Framework 3.16.1
- PostgreSQL database
- File storage: Local filesystem or S3 (django-storages + boto3)
- OpenAPI/Swagger documentation (drf-spectacular)

**Frontend:**
- Vue 3.5.13 (Composition API + TypeScript)
- Vuetify 3.10.3 (Material Design components)
- Vite 7.1.11 (build tool)
- Auto-generated TypeScript API client (openapi-generator-cli)

### Current Step Model Structure
**Location:** `cookbook/models.py:959`

```python
class Step(models.Model):
    name = models.CharField(max_length=128, default='', blank=True)
    instruction = models.TextField(blank=True)
    ingredients = models.ManyToManyField(Ingredient, blank=True)
    time = models.IntegerField(default=0, blank=True)
    order = models.IntegerField(default=0)
    file = models.ForeignKey('UserFile', on_delete=models.PROTECT, null=True, blank=True)  # Currently for images
    show_as_header = models.BooleanField(default=True)
    show_ingredients_table = models.BooleanField(default=True)
    search_vector = SearchVectorField(null=True)
    step_recipe = models.ForeignKey('Recipe', on_delete=models.PROTECT, null=True, blank=True)
    space = models.ForeignKey(Space, on_delete=models.CASCADE)
```

### Current File Handling
**Location:** `cookbook/models.py:1568`

**UserFile Model:**
- Supports any file type with validation (`is_file_type_allowed()`)
- Tracks file size and enforces storage quotas per space
- Provides download and preview URLs
- S3-compatible storage backend
- Current limitation: Step.file is designed for images (single file)

**Key Components:**
- **Backend:** UserFileSerializer, UserFileViewSet
- **Frontend:** StepEditor.vue, StepView.vue, StepMarkdownEditor.vue

---

## Implementation Plan

### Phase 1: Backend - Database Schema

#### Option A: Add Voice File Field to Step Model (Recommended)
**Pros:**
- Simple, leverages existing UserFile infrastructure
- Consistent with current file handling pattern
- No additional joins required

**Cons:**
- Limits to one voice message per step
- Less flexible for future multi-audio support

**Implementation:**
```python
# Add to Step model (cookbook/models.py)
voice_file = models.ForeignKey(
    'UserFile',
    on_delete=models.SET_NULL,
    null=True,
    blank=True,
    related_name='voice_steps'
)
```

**Migration Steps:**
1. Create Django migration: `python manage.py makemigrations`
2. Apply migration: `python manage.py migrate`
3. Update Step serializer to include `voice_file` field

#### Option B: Many-to-Many Relationship (Future-Proof)
**Pros:**
- Supports multiple voice messages per step
- More flexible for future features (multiple takes, languages)

**Cons:**
- More complex queries
- Overkill for MVP

**Implementation:**
```python
# Add to Step model
voice_files = models.ManyToManyField(
    'UserFile',
    blank=True,
    related_name='voice_steps'
)
```

**Recommendation:** Start with Option A (single voice file), can migrate to Option B later if needed.

---

### Phase 2: Backend - API Updates

#### 2.1 Update StepSerializer
**Location:** `cookbook/serializer.py:1047`

**Changes:**
```python
class StepSerializer(WritableNestedModelSerializer, ExtendedRecipeMixin):
    ingredients = IngredientSerializer(many=True)
    instructions_markdown = serializers.SerializerMethodField('get_instructions_markdown')
    file = UserFileViewSerializer(allow_null=True, required=False)
    voice_file = UserFileViewSerializer(allow_null=True, required=False)  # NEW
    step_recipe_data = serializers.SerializerMethodField('get_step_recipe_data')

    class Meta:
        model = Step
        fields = ('id', 'name', 'instruction', 'ingredients', 'instructions_markdown',
                  'time', 'order', 'show_as_header', 'file', 'voice_file',  # NEW FIELD
                  'step_recipe', 'step_recipe_data', 'numrecipe', 'show_ingredients_table')
```

#### 2.2 Voice File Validation
**Location:** `cookbook/models.py` (UserFile model)

**Add Audio File Type Validation:**
```python
# Add to UserFile model or create helper function
ALLOWED_AUDIO_FORMATS = ['.mp3', '.wav', '.ogg', '.webm', '.m4a', '.aac']

def is_audio_file(self):
    """Check if file is an audio file"""
    ext = os.path.splitext(self.name)[1].lower()
    return ext in ALLOWED_AUDIO_FORMATS
```

**Update File Validation:**
- Add audio formats to allowed file types
- Set reasonable file size limits (e.g., 10MB max for voice messages)
- Consider compression/transcoding for large files

#### 2.3 API Endpoint Considerations
- No new endpoints needed (use existing StepViewSet)
- Voice file uploaded via existing `/api/user_file/` endpoint
- Step PATCH request includes `voice_file` ID reference
- GET `/api/step/{id}/` returns voice_file data with download URL

---

### Phase 3: Frontend - TypeScript API Client

#### 3.1 Regenerate OpenAPI Client
**After backend changes:**
```bash
cd vue3
npm run generate-api-client
```

This will auto-generate TypeScript interfaces with the new `voice_file` field:
```typescript
interface Step {
    id: number;
    name: string;
    instruction: string;
    file?: UserFile;
    voice_file?: UserFile;  // NEW
    // ... other fields
}
```

---

### Phase 4: Frontend - Voice Recording Component

#### 4.1 Create VoiceRecorder.vue Component
**Location:** `vue3/src/components/inputs/VoiceRecorder.vue`

**Features:**
- Record button with visual feedback (recording indicator)
- Stop recording button
- Playback controls (play, pause, seek)
- Delete/Re-record functionality
- Audio waveform visualization (optional, nice-to-have)
- Recording time display
- Upload status indicator

**Technologies:**
- **MediaRecorder API** (browser native audio recording)
- **Web Audio API** (for waveform visualization, optional)
- **Vuetify components:** v-btn, v-progress-circular, v-slider, v-icon

**Component Interface:**
```typescript
interface VoiceRecorderProps {
    existingFile?: UserFile | null;  // Pre-existing voice file
    disabled?: boolean;
}

interface VoiceRecorderEmits {
    (e: 'file-uploaded', file: UserFile): void;
    (e: 'file-deleted'): void;
}
```

**Key Methods:**
```typescript
// Start recording
async startRecording(): Promise<void>

// Stop recording and return audio blob
async stopRecording(): Promise<Blob>

// Upload audio blob to backend
async uploadAudio(blob: Blob): Promise<UserFile>

// Delete voice file
async deleteVoiceFile(fileId: number): Promise<void>
```

**Browser Compatibility:**
- MediaRecorder API: Chrome 47+, Firefox 25+, Safari 14+
- Fallback: Show "Recording not supported" message for older browsers

#### 4.2 Audio Format Considerations
**Recording Format:**
- **WebM** (opus codec) - Best browser support, good compression
- **MP4** (aac codec) - Safari fallback
- Auto-detect best format based on browser

**Implementation:**
```typescript
let mimeType = 'audio/webm;codecs=opus';
if (!MediaRecorder.isTypeSupported(mimeType)) {
    mimeType = 'audio/mp4;codecs=aac';  // Safari fallback
}
```

---

### Phase 5: Frontend - StepEditor Integration

#### 5.1 Update StepEditor.vue
**Location:** `vue3/src/components/inputs/StepEditor.vue`

**Changes:**
1. Import and add VoiceRecorder component
2. Add voice file state to step data
3. Handle voice file upload events
4. Display voice file status/name when present

**UI Layout:**
```
┌─────────────────────────────────────────┐
│ Step Name: [_______________]            │
│                                         │
│ Instruction (Markdown):                 │
│ ┌─────────────────────────────────────┐ │
│ │ [Markdown Editor]                   │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Voice Message:                          │
│ ┌─────────────────────────────────────┐ │
│ │ [VoiceRecorder Component]           │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ Image/File: [Choose File]              │
│                                         │
│ Time: [__] min    [ ] Show as header    │
└─────────────────────────────────────────┘
```

**Code Changes:**
```typescript
<template>
  <v-card>
    <!-- Existing fields -->

    <!-- NEW: Voice Message Section -->
    <v-card-subtitle>{{ $t('Voice Message') }}</v-card-subtitle>
    <VoiceRecorder
      :existing-file="step.voice_file"
      @file-uploaded="handleVoiceFileUploaded"
      @file-deleted="handleVoiceFileDeleted"
    />

    <!-- Existing fields continue -->
  </v-card>
</template>

<script setup lang="ts">
function handleVoiceFileUploaded(file: UserFile) {
    step.voice_file = file;
}

function handleVoiceFileDeleted() {
    step.voice_file = null;
}
</script>
```

---

### Phase 6: Frontend - StepView Integration

#### 6.1 Update StepView.vue
**Location:** `vue3/src/components/display/StepView.vue`

**Changes:**
1. Display voice file indicator (icon/badge) when present
2. Add audio player controls
3. Position audio player appropriately (below instruction, above image)

**UI Layout:**
```
┌─────────────────────────────────────────┐
│ 1. Step Name                       [✓]  │  <- Checkbox for completion
│                                         │
│ Step instructions in markdown...       │
│                                         │
│ ┌─────────────────────────────────────┐ │
│ │ 🎤 Voice Message                    │ │  <- NEW
│ │ [►] ──────○────── 1:23 / 2:45       │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ [Step Image]                            │
│                                         │
│ Ingredients:                            │
│ • Ingredient 1                          │
│ • Ingredient 2                          │
└─────────────────────────────────────────┘
```

**Audio Player Features:**
- Play/Pause button
- Playback progress bar (seekable)
- Current time / Total duration display
- Volume control (optional)
- Playback speed control (0.75x, 1x, 1.25x, 1.5x) (optional, nice-to-have)

**Component Implementation:**
```typescript
<template>
  <v-card>
    <!-- Existing instruction text -->

    <!-- NEW: Voice Message Player -->
    <v-card v-if="step.voice_file" class="ma-2">
      <v-card-title class="text-subtitle-2">
        <v-icon>mdi-microphone</v-icon>
        {{ $t('Voice Message') }}
      </v-card-title>
      <v-card-text>
        <audio
          ref="audioPlayer"
          :src="step.voice_file.download_url"
          @timeupdate="updateProgress"
          @loadedmetadata="setDuration"
        />

        <div class="d-flex align-center">
          <v-btn
            icon
            @click="togglePlay"
            :disabled="!audioLoaded"
          >
            <v-icon>{{ isPlaying ? 'mdi-pause' : 'mdi-play' }}</v-icon>
          </v-btn>

          <v-slider
            v-model="currentTime"
            :max="duration"
            :step="0.1"
            hide-details
            class="mx-2"
            @update:model-value="seek"
          />

          <span class="text-caption">
            {{ formatTime(currentTime) }} / {{ formatTime(duration) }}
          </span>
        </div>
      </v-card-text>
    </v-card>

    <!-- Existing image and ingredients -->
  </v-card>
</template>

<script setup lang="ts">
import { ref, onUnmounted } from 'vue';

const audioPlayer = ref<HTMLAudioElement>();
const isPlaying = ref(false);
const currentTime = ref(0);
const duration = ref(0);
const audioLoaded = ref(false);

function togglePlay() {
    if (!audioPlayer.value) return;

    if (isPlaying.value) {
        audioPlayer.value.pause();
    } else {
        audioPlayer.value.play();
    }
    isPlaying.value = !isPlaying.value;
}

function updateProgress(event: Event) {
    const audio = event.target as HTMLAudioElement;
    currentTime.value = audio.currentTime;
}

function setDuration(event: Event) {
    const audio = event.target as HTMLAudioElement;
    duration.value = audio.duration;
    audioLoaded.value = true;
}

function seek(value: number) {
    if (audioPlayer.value) {
        audioPlayer.value.currentTime = value;
    }
}

function formatTime(seconds: number): string {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${secs.toString().padStart(2, '0')}`;
}

onUnmounted(() => {
    if (audioPlayer.value) {
        audioPlayer.value.pause();
    }
});
</script>
```

---

### Phase 7: UI/UX Enhancements

#### 7.1 Visual Indicators
- **Recipe Card:** Show microphone icon badge if any step has voice
- **Steps Overview:** Add voice indicator column in ingredient table
- **Step List:** Highlight steps with voice messages

#### 7.2 Accessibility
- ARIA labels for all audio controls
- Keyboard navigation support (Space = play/pause, arrows = seek)
- Screen reader announcements for recording state
- Alternative text descriptions

#### 7.3 Mobile Considerations
- Touch-friendly button sizes (min 48x48px)
- Simplified controls on small screens
- Request microphone permissions properly
- Test on iOS Safari (WebKit restrictions)

#### 7.4 Loading States
- Show spinner during audio upload
- Display progress for large file uploads
- Disable controls during processing
- Error handling with user-friendly messages

---

### Phase 8: Internationalization (i18n)

#### 8.1 Add Translation Keys
**Location:** `vue3/src/locales/*.json`

**New Translation Keys:**
```json
{
    "Voice Message": "Voice Message",
    "Record Voice": "Record Voice",
    "Stop Recording": "Stop Recording",
    "Delete Voice Message": "Delete Voice Message",
    "Play": "Play",
    "Pause": "Pause",
    "Recording...": "Recording...",
    "Uploading voice message...": "Uploading voice message...",
    "Voice message uploaded successfully": "Voice message uploaded successfully",
    "Failed to upload voice message": "Failed to upload voice message",
    "Recording not supported": "Voice recording is not supported in your browser",
    "Microphone permission denied": "Microphone access was denied. Please enable microphone permissions in your browser settings."
}
```

**Languages to Support:**
- English (en)
- German (de)
- French (fr)
- Spanish (es)
- Italian (it)
- Portuguese (pt)
- Dutch (nl)
- And other existing Tandoor languages

---

## Technical Considerations

### Security
1. **File Type Validation:**
   - Server-side validation of audio MIME types
   - Reject non-audio files attempting to masquerade
   - Use magic number checking (not just file extension)

2. **File Size Limits:**
   - Set reasonable max size (e.g., 10MB per voice message)
   - Enforce at both client and server level
   - Consider space storage quotas

3. **Permissions:**
   - Respect existing space-based permissions
   - Only file creator or space admins can delete voice files
   - Voice files inherit step/recipe permissions

4. **XSS Prevention:**
   - Serve audio files with correct Content-Type headers
   - Use Content-Security-Policy headers
   - No inline scripts in audio player

### Performance
1. **Audio Compression:**
   - Use opus codec (best quality/size ratio)
   - Consider server-side transcoding to optimize file sizes
   - Lazy load audio files (don't preload all on page load)

2. **Caching:**
   - Set appropriate Cache-Control headers for audio files
   - Use browser caching for repeated plays
   - Consider CDN for large deployments

3. **Streaming:**
   - Use HTTP range requests for seeking
   - Enable partial content delivery (Accept-Ranges: bytes)
   - Stream audio instead of full download before play

### Browser Compatibility
| Feature | Chrome | Firefox | Safari | Edge |
|---------|--------|---------|--------|------|
| MediaRecorder API | ✓ 47+ | ✓ 25+ | ✓ 14+ | ✓ 79+ |
| WebM/Opus | ✓ | ✓ | ✗ | ✓ |
| MP4/AAC | ✓ | ✓ | ✓ | ✓ |
| Audio Element | ✓ | ✓ | ✓ | ✓ |

**Fallback Strategy:**
- Detect MediaRecorder support on component mount
- Show "Not supported" message with link to compatible browsers
- Gracefully degrade: Users can still see/edit text instructions

### Storage Considerations
1. **File System:**
   - Voice files stored in `/media/files/` (existing UserFile location)
   - Namespaced by space_id for multi-tenant isolation

2. **S3 Integration:**
   - Existing S3 setup should work out-of-box
   - Configure lifecycle policies (e.g., delete old unused voice files)
   - Use presigned URLs for secure audio delivery

3. **Database:**
   - Voice file metadata stored in UserFile table
   - ForeignKey reference from Step to UserFile
   - Index on Step.voice_file for faster queries

---

## Testing Strategy

### Backend Tests

#### Unit Tests
**Location:** `cookbook/tests/api/test_api_step.py`

**Test Cases:**
```python
# 1. Test voice file field in Step model
def test_step_with_voice_file(u1_s1, u1_s2, obj_1, obj_2):
    # Create voice file
    audio_file = UserFile.objects.create(...)
    # Create step with voice file
    step = Step.objects.create(..., voice_file=audio_file)
    assert step.voice_file == audio_file

# 2. Test voice file serialization
def test_step_serializer_includes_voice_file():
    # Assert voice_file field present in serializer output

# 3. Test voice file deletion behavior
def test_voice_file_set_null_on_delete():
    # Delete voice file, step.voice_file should be null

# 4. Test voice file permissions
def test_voice_file_respects_space_permissions():
    # User from space_1 cannot access voice_file from space_2

# 5. Test voice file size validation
def test_voice_file_size_limit():
    # Upload large file, should be rejected
```

#### Integration Tests
```python
# 1. Test full recipe creation with voice files
def test_create_recipe_with_voice_steps():
    # POST /api/recipe/ with steps containing voice_files

# 2. Test voice file upload workflow
def test_upload_voice_file_and_attach_to_step():
    # POST /api/user_file/ (upload audio)
    # PATCH /api/step/{id}/ (attach voice_file)

# 3. Test recipe import/export with voice files
def test_recipe_export_includes_voice_files():
    # Export recipe, voice files should be included
```

### Frontend Tests

#### Unit Tests (Vitest)
**Location:** `vue3/tests/unit/`

**Test Cases:**
```typescript
// VoiceRecorder.vue tests
describe('VoiceRecorder', () => {
    it('starts recording when record button clicked', async () => {
        // Mock MediaRecorder
        // Click record button
        // Assert recording state is true
    });

    it('stops recording and uploads audio', async () => {
        // Start recording
        // Stop recording
        // Assert API call made to upload audio
    });

    it('displays existing voice file', () => {
        // Pass existing file as prop
        // Assert file name displayed
    });

    it('deletes voice file when delete button clicked', async () => {
        // Click delete button
        // Assert API call made to delete file
        // Assert file-deleted event emitted
    });
});

// StepEditor.vue tests
describe('StepEditor with voice', () => {
    it('displays VoiceRecorder component', () => {
        // Mount StepEditor
        // Assert VoiceRecorder present
    });

    it('updates step voice_file on upload', async () => {
        // Emit file-uploaded event from VoiceRecorder
        // Assert step.voice_file updated
    });
});

// StepView.vue tests
describe('StepView with voice', () => {
    it('displays audio player when voice_file present', () => {
        // Pass step with voice_file
        // Assert audio element present
    });

    it('plays audio when play button clicked', async () => {
        // Click play button
        // Assert audio.play() called
    });
});
```

#### E2E Tests (Playwright/Cypress)
**Location:** `vue3/tests/e2e/`

**Test Scenarios:**
```typescript
// Full workflow test
it('records voice message for step', () => {
    // 1. Login
    // 2. Open recipe editor
    // 3. Add new step
    // 4. Click record button
    // 5. Wait 2 seconds
    // 6. Click stop
    // 7. Wait for upload
    // 8. Save recipe
    // 9. Navigate to recipe view
    // 10. Assert voice player visible
    // 11. Click play
    // 12. Assert audio playing
});

// Delete voice message test
it('deletes voice message', () => {
    // 1. Open step with voice message
    // 2. Click delete button
    // 3. Confirm deletion
    // 4. Assert voice player hidden
});
```

### Manual Testing Checklist
- [ ] Record voice message in Chrome (Windows/Mac/Linux)
- [ ] Record voice message in Firefox
- [ ] Record voice message in Safari (Mac/iOS)
- [ ] Record voice message in Edge
- [ ] Test on mobile devices (Android/iOS)
- [ ] Test with microphone permission denied
- [ ] Test with no microphone connected
- [ ] Test file size limits
- [ ] Test storage quota limits
- [ ] Test audio playback on all browsers
- [ ] Test seeking in audio player
- [ ] Test volume controls
- [ ] Test with screen reader (accessibility)
- [ ] Test with keyboard navigation only
- [ ] Test recipe export/import with voice files
- [ ] Test voice file deletion
- [ ] Test permissions (different users/spaces)

---

## Migration Path

### Database Migration
1. **Create Migration:**
   ```bash
   python manage.py makemigrations
   ```

2. **Review Migration:**
   - Check generated SQL
   - Ensure backward compatibility
   - Verify indexes created

3. **Apply Migration:**
   ```bash
   python manage.py migrate
   ```

4. **Rollback Plan:**
   - Voice file is nullable, so safe to rollback
   - Existing steps unaffected
   - Can drop column if needed

### API Versioning
- No breaking changes (voice_file is optional field)
- Existing API consumers continue to work
- New field ignored by older clients

### Frontend Deployment
- Update OpenAPI client first
- Deploy frontend with backward compatibility
- Progressive enhancement: Feature available when backend supports it

---

## Future Enhancements

### Phase 2 Features (Post-MVP)
1. **Multiple Voice Messages per Step:**
   - Switch to many-to-many relationship
   - Support for multiple takes/versions
   - Language-specific voice messages (multi-language recipes)

2. **Audio Processing:**
   - Server-side transcoding (standardize format)
   - Automatic speech-to-text transcription
   - Noise reduction/enhancement
   - Automatic volume normalization

3. **Advanced Playback:**
   - Continuous playback across steps
   - Auto-play next step's voice message
   - Background playback while scrolling
   - Picture-in-picture audio controls

4. **Voice Message Timeline:**
   - Timestamp markers in audio
   - Jump to specific ingredient mentions
   - Synchronized text highlighting during playback

5. **Social Features:**
   - Voice comments from other users
   - Voice Q&A on recipe steps
   - Voice tips from community

6. **Offline Support:**
   - Cache audio files in service worker
   - Progressive Web App (PWA) enhancements
   - Download recipes with audio for offline use

7. **Analytics:**
   - Track voice message engagement
   - Measure completion rates for steps with voice
   - A/B testing: voice vs text-only steps

8. **AI Integration:**
   - Auto-generate voice from text instructions (TTS)
   - Voice cloning for consistent recipe narration
   - Language translation of voice messages
   - Smart audio summarization

---

## Open Questions & Decisions Needed

### Technical Decisions
- [ ] **Audio Format:** WebM-Opus primary? MP4-AAC fallback? Both?
- [ ] **Max File Size:** 10MB reasonable? Adjust based on testing?
- [ ] **Storage Strategy:** Keep local files? Migrate to S3 only?
- [ ] **Transcoding:** Server-side transcoding needed initially?
- [ ] **Progressive Enhancement:** Hide feature in unsupported browsers or show with warning?

### UX Decisions
- [ ] **Default State:** Voice section collapsed or expanded in StepEditor?
- [ ] **Placement:** Voice player above or below step image in StepView?
- [ ] **Icon:** Microphone, speaker, or headphones icon for indicator?
- [ ] **Auto-play:** Should voice messages auto-play when step is viewed?
- [ ] **Waveform:** Show audio waveform visualization or simple progress bar?

### Product Decisions
- [ ] **Permissions:** Should all users be able to record or only recipe creators?
- [ ] **Moderation:** Content moderation needed for voice messages?
- [ ] **Privacy:** GDPR considerations for voice recordings?
- [ ] **Premium Feature:** Free for all or premium/pro feature?

---

## Resources & References

### Documentation
- **MediaRecorder API:** https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder
- **Web Audio API:** https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API
- **HTML Audio Element:** https://developer.mozilla.org/en-US/docs/Web/HTML/Element/audio
- **Django File Uploads:** https://docs.djangoproject.com/en/5.0/topics/http/file-uploads/
- **DRF File Fields:** https://www.django-rest-framework.org/api-guide/fields/#filefield

### Libraries to Consider
- **wavesurfer.js:** Audio waveform visualization (https://wavesurfer-js.org/)
- **recordrtc:** Cross-browser audio recording (https://github.com/muaz-khan/RecordRTC)
- **howler.js:** Advanced audio playback library (https://howlerjs.com/)
- **vue-audio-better:** Vue audio player component (evaluate if useful)

### Tandoor-Specific Files
**Backend:**
- `cookbook/models.py` - Step and UserFile models
- `cookbook/serializer.py` - StepSerializer, UserFileSerializer
- `cookbook/views/api.py` - StepViewSet, UserFileViewSet
- `cookbook/tests/api/test_api_step.py` - Step API tests

**Frontend:**
- `vue3/src/components/inputs/StepEditor.vue` - Step editing UI
- `vue3/src/components/display/StepView.vue` - Step display UI
- `vue3/src/openapi/` - Auto-generated API client
- `vue3/src/locales/` - i18n translation files

---

## Timeline Estimate (For Reference Only)

**Note:** This is a rough estimate for planning purposes. Actual implementation time may vary based on:
- Developer experience with Django, Vue, and Web APIs
- Code review and iteration cycles
- Testing thoroughness
- Unexpected technical challenges

### Development Phases
1. **Backend (Database + API):** 2-3 days
   - Database migration
   - Serializer updates
   - API testing

2. **Frontend - VoiceRecorder Component:** 3-4 days
   - MediaRecorder integration
   - UI/UX implementation
   - Upload functionality
   - Error handling

3. **Frontend - StepEditor Integration:** 1-2 days
   - Component integration
   - State management
   - Form validation

4. **Frontend - StepView Integration:** 2-3 days
   - Audio player implementation
   - Playback controls
   - Mobile optimization

5. **Testing:** 2-3 days
   - Unit tests
   - Integration tests
   - E2E tests
   - Manual testing

6. **Internationalization:** 1 day
   - Translation keys
   - Multi-language testing

7. **Documentation & Deployment:** 1 day
   - Update user docs
   - Migration guide
   - Deployment checklist

**Total Estimate:** 12-17 days (assuming full-time development)

---

## Success Metrics

### Technical Metrics
- [ ] Voice file upload success rate > 95%
- [ ] Audio playback works on 99% of supported browsers
- [ ] Average upload time < 10 seconds for typical voice message
- [ ] Zero SQL query regressions (voice field doesn't slow down step queries)
- [ ] Voice file storage stays within space quotas

### User Metrics (Post-Launch)
- [ ] % of recipes with voice messages (adoption rate)
- [ ] % of users who record voice messages
- [ ] % of users who play voice messages
- [ ] Average voice message duration
- [ ] Voice message playback completion rate
- [ ] User feedback score (surveys)

### Quality Metrics
- [ ] Zero security vulnerabilities
- [ ] 100% test coverage for new code
- [ ] Zero critical bugs in production
- [ ] Accessibility score: WCAG 2.1 AA compliant
- [ ] Performance: No page load time regressions

---

## Change Log

| Date | Version | Changes | Author |
|------|---------|---------|--------|
| 2026-01-01 | 0.1 | Initial planning document created | AI Assistant |
|  |  |  |  |
|  |  |  |  |

---

## Notes & Observations

*Use this section for implementation notes, gotchas, lessons learned, etc.*

---

**END OF WORK LOG**

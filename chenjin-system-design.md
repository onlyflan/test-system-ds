# 沉浸 (Chénjìn) — Technical Blueprint
> Senior Architect Edition · Build-ready · 1 dev / small team

---

## 0. Refined Concept (Góc nhìn kỹ thuật)

**Core value một dòng:**  
User consume nội dung Trung thật → highlight từ không hiểu → nhận giải thích đúng context → hệ thống tự lên lịch ôn lại theo SRS.

**Input / Output của toàn hệ thống:**

| | Mô tả |
|---|---|
| **Input** | Content (tweet/bài báo/hội thoại) + hành vi user (highlight, save, review) |
| **Output chính** | Contextual explanation (tiếng Việt, culture-aware) + Memory Capsule + Review schedule |
| **Output phụ** | Behavior analytics, learning progress map |

**Ràng buộc kỹ thuật cứng:**
- 1 dev build được trong 4 tuần MVP
- Explanation phải xuất hiện trong ≤2s (on-device path)
- Hoạt động offline (degraded mode)
- Chi phí LLM kiểm soát được qua cache

---

## 1. System Design — High Level

```
┌─────────────────────────────────────────────────────────────────┐
│                        ANDROID APP                              │
│                                                                 │
│  ContentFeed ──► ReadingScreen ──► HighlightGesture             │
│                        │                                        │
│            ┌───────────┴───────────┐                           │
│            ▼                       ▼                            │
│     OnDeviceEngine           CloudEngine (async)                │
│   (Qwen2.5-3B Q4)           (Gemini API, user key)             │
│            │                       │                            │
│     ~1-2s response            ~3-8s response                    │
│            └───────────┬───────────┘                           │
│                        ▼                                        │
│                ExplanationCache (Room DB)                       │
│                        │                                        │
│                  SaveCapsule?                                    │
│                        ▼                                        │
│              MemoryCapsuleStore (Room DB)                        │
│                        │                                        │
│              ReviewScheduler (FSRS algo)                        │
└─────────────┬───────────────────────────────────────────────────┘
              │  REST API (khi online)
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SPRING BOOT BACKEND                          │
│                                                                 │
│  ContentAPI ── ExplainCacheAPI ── UserAPI ── SyncAPI            │
│       │                │                          │             │
│  ContentStore    ExplainCache              UserCapsuleStore     │
│  (curated)      (Redis/PG)                (PostgreSQL)          │
│                                                                 │
│  [Background Jobs]                                              │
│  ContentIngestion ── CacheWarmup ── ReviewReminder              │
│       │                  │                                      │
│  Python NLP sidecar    Gemini API (server-side warmup)          │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATABASE LAYER                             │
│                                                                 │
│  PostgreSQL (main) + pgvector (embeddings) + Redis (cache)      │
└─────────────────────────────────────────────────────────────────┘
```

**Data flow — happy path (user highlight từ):**

```
1. User bôi đen "甲方" trong tweet
2. Android: debounce 300ms → extract (word="甲方", context=100 chars xung quanh)
3. Android: check Room cache → miss
4. Android: ĐỒNG THỜI gọi:
   A) OnDeviceEngine.explain(word, context) — synchronous coroutine
   B) CloudEngine.explainAsync(word, context, apiKey) — background coroutine
5. Path A (~1.2s): kết quả về → update UI state → hiển thị bubble "Quick"
6. Path B (~4s): kết quả về → so sánh với Path A
   → nếu bubble còn mở: update in-place, badge "Enhanced"
   → nếu đã đóng: save enhanced version vào cache
7. User nhấn ⊕ Save → tạo MemoryCapsule → sync lên backend khi online
8. Backend: background job tính FSRS due_date → schedule review notification
```

---

## 2. Kiến trúc chi tiết — Backend

### Module structure (Spring Boot)

```
chenjin-backend/
├── src/main/java/com/chenjin/
│   ├── content/                    # Content management
│   │   ├── ContentController.java
│   │   ├── ContentService.java
│   │   ├── ContentRepository.java
│   │   └── domain/
│   │       ├── ContentItem.java    # Entity
│   │       └── TokenMap.java       # Tokenized representation
│   │
│   ├── explain/                    # Explanation cache
│   │   ├── ExplainCacheController.java
│   │   ├── ExplainCacheService.java
│   │   ├── ExplainCacheRepository.java
│   │   └── domain/
│   │       ├── ExplainCache.java
│   │       └── ExplainRequest.java # DTO
│   │
│   ├── capsule/                    # Memory capsules
│   │   ├── CapsuleController.java
│   │   ├── CapsuleService.java
│   │   ├── CapsuleRepository.java
│   │   └── domain/
│   │       ├── MemoryCapsule.java
│   │       └── LearningState.java  # FSRS state
│   │
│   ├── user/                       # Auth + user profile
│   │   ├── AuthController.java
│   │   ├── UserService.java
│   │   └── domain/User.java
│   │
│   ├── jobs/                       # Background tasks
│   │   ├── ContentIngestionJob.java
│   │   ├── CacheWarmupJob.java
│   │   └── ReviewReminderJob.java
│   │
│   └── shared/                     # Cross-cutting
│       ├── config/SecurityConfig.java
│       ├── config/RedisConfig.java
│       └── exception/GlobalExceptionHandler.java
│
└── src/main/resources/
    └── application.yml
```

### API Design — các endpoint cần thiết

```yaml
# Content
GET  /api/v1/feed?topic=humor&offset=0&limit=10
GET  /api/v1/content/{id}

# Explain cache (server-side)
GET  /api/v1/explain/cache?word=甲方&source_type=weibo
POST /api/v1/explain/cache          # admin: pre-warm entry

# Capsules (sync)
POST /api/v1/capsules               # save capsule
GET  /api/v1/capsules/due           # get due for review today
PUT  /api/v1/capsules/{id}/review   # submit review result

# Auth
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
```

### Luồng xử lý request tiêu biểu — POST /capsules

```java
// CapsuleController.java
@PostMapping
public ResponseEntity<CapsuleResponse> saveCapsule(
    @RequestBody SaveCapsuleRequest req,
    @AuthenticationPrincipal UserPrincipal user
) {
    MemoryCapsule capsule = capsuleService.save(user.getId(), req);
    return ResponseEntity.ok(CapsuleResponse.from(capsule));
}

// CapsuleService.java
public MemoryCapsule save(UUID userId, SaveCapsuleRequest req) {
    // 1. Tạo capsule entity
    MemoryCapsule capsule = MemoryCapsule.builder()
        .userId(userId)
        .surfaceForm(req.word())
        .contextWindow(req.contextWindow())       // 100 chars
        .sourceContentId(req.contentId())
        .explanation(req.explanation())
        .register(req.register())
        .topics(req.topics())
        .build();

    // 2. Khởi tạo FSRS learning state
    LearningState state = LearningState.initial(capsule.getId());
    capsule.setLearningState(state);

    // 3. Save
    return capsuleRepository.save(capsule);
    // FSRS due_date được tính trong LearningState.initial() = now + 1 day
}
```

### CacheWarmup Job — quan trọng nhất về performance

```java
@Component
public class CacheWarmupJob {

    @Scheduled(fixedDelay = 3_600_000) // mỗi 1 giờ
    public void warmUpNewContent() {
        // Lấy content vừa được schedule push trong 2h tới
        List<ContentItem> upcoming = contentRepo.findScheduledSoon(Duration.ofHours(2));
        
        for (ContentItem item : upcoming) {
            List<String> topWords = extractTopWords(item, 20); // top 20 từ khó
            
            for (String word : topWords) {
                String cacheKey = word + "::" + item.getSourceType();
                
                if (!explainCache.exists(cacheKey)) {
                    // Gọi Gemini từ server-side (dùng server API key, không phải user key)
                    String explanation = geminiService.explain(word, item.getContextFor(word));
                    explainCache.put(cacheKey, explanation, Duration.ofDays(7));
                }
            }
        }
    }

    private List<String> extractTopWords(ContentItem item, int n) {
        // Gọi Python NLP sidecar để tokenize + lấy từ có HSK level >= 4
        // hoặc từ không có trong top 3000 từ phổ thông
        return nlpSidecarClient.getHardWords(item.getText(), n);
    }
}
```

---

## 3. Kiến trúc chi tiết — Android

### Clean Architecture + MVI

```
app/
├── data/
│   ├── local/
│   │   ├── db/AppDatabase.kt          # Room database
│   │   ├── dao/CapsuleDao.kt
│   │   ├── dao/ExplainCacheDao.kt
│   │   ├── dao/ContentDao.kt
│   │   └── prefs/UserPreferences.kt   # DataStore
│   ├── remote/
│   │   ├── api/BackendApi.kt           # Retrofit interface
│   │   ├── api/GeminiApi.kt
│   │   └── dto/                        # Request/Response DTOs
│   └── repository/
│       ├── ContentRepository.kt
│       ├── CapsuleRepository.kt
│       └── ExplainRepository.kt        # Orchestrates on-device + cloud
│
├── domain/
│   ├── model/
│   │   ├── ContentItem.kt
│   │   ├── MemoryCapsule.kt
│   │   ├── Explanation.kt
│   │   └── ReviewItem.kt
│   ├── usecase/
│   │   ├── ExplainWordUseCase.kt       # Core use case
│   │   ├── SaveCapsuleUseCase.kt
│   │   ├── GetFeedUseCase.kt
│   │   └── GetDueReviewsUseCase.kt
│   └── srs/
│       └── FSRSAlgorithm.kt            # Pure Kotlin, no deps
│
├── presentation/
│   ├── feed/
│   │   ├── FeedViewModel.kt
│   │   ├── FeedScreen.kt
│   │   └── FeedState.kt                # MVI state
│   ├── reading/
│   │   ├── ReadingViewModel.kt
│   │   ├── ReadingScreen.kt
│   │   ├── ReadingState.kt
│   │   └── components/
│   │       ├── SelectableText.kt       # Custom highlight component
│   │       └── ExplanationBubble.kt
│   ├── review/
│   │   ├── ReviewViewModel.kt
│   │   └── ReviewScreen.kt
│   └── settings/
│       └── SettingsScreen.kt           # API key input, model download
│
└── ai/
    ├── ondevice/
    │   ├── OnDeviceEngine.kt           # MediaPipe wrapper
    │   └── OnDeviceModelManager.kt     # Download, version check
    └── cloud/
        └── CloudEngine.kt              # Gemini API caller
```

### State management — MVI pattern cho ReadingScreen

```kotlin
// ReadingState.kt
data class ReadingState(
    val content: ContentItem? = null,
    val isLoading: Boolean = false,
    val selectedText: String = "",
    val selectionRange: TextRange? = null,
    val explanation: ExplanationUiState = ExplanationUiState.Idle,
)

sealed class ExplanationUiState {
    object Idle : ExplanationUiState()
    object Loading : ExplanationUiState()
    data class QuickResult(val explanation: Explanation) : ExplanationUiState()
    data class EnhancedResult(val explanation: Explanation) : ExplanationUiState()
    data class Error(val message: String) : ExplanationUiState()
}

// ReadingIntent.kt
sealed class ReadingIntent {
    data class TextSelected(val text: String, val range: TextRange) : ReadingIntent()
    object DismissExplanation : ReadingIntent()
    data class SaveCapsule(val explanation: Explanation) : ReadingIntent()
    data class ReviewResult(val rating: Int) : ReadingIntent()
}
```

### ExplainWordUseCase — trái tim của app

```kotlin
class ExplainWordUseCase @Inject constructor(
    private val onDeviceEngine: OnDeviceEngine,
    private val cloudEngine: CloudEngine,
    private val explainRepository: ExplainRepository,
) {
    // Trả về Flow để ViewModel observe từng update
    fun explain(word: String, context: String, sourceType: String): Flow<ExplanationResult> = flow {

        // Step 1: Check cache trước (Room local)
        val cached = explainRepository.getCached(word, sourceType)
        if (cached != null) {
            emit(ExplanationResult.Enhanced(cached)) // Cache = always enhanced
            return@flow
        }

        // Step 2: Phát signal loading
        emit(ExplanationResult.Loading)

        // Step 3: On-device inference
        val onDeviceResult = onDeviceEngine.explain(
            prompt = buildPrompt(word, context, sourceType)
        )
        emit(ExplanationResult.Quick(onDeviceResult))

        // Step 4: Cloud call trong coroutine riêng — không block
        val cloudResult = runCatching {
            cloudEngine.explain(word, context, sourceType)
        }.getOrNull()

        if (cloudResult != null) {
            // Cache lại để lần sau dùng ngay
            explainRepository.cache(word, sourceType, cloudResult)
            emit(ExplanationResult.Enhanced(cloudResult))
        }
        // Nếu cloud fail → giữ on-device result, không emit error
    }

    private fun buildPrompt(word: String, context: String, sourceType: String): String = """
        Bạn là trợ lý giải nghĩa tiếng Trung cho người Việt học.
        Từ cần giải thích: "$word"
        Ngữ cảnh (đây là câu/đoạn văn thực tế từ $sourceType):
        "$context"
        
        Giải thích NGẮN (tối đa 3 câu tiếng Việt):
        1. Nghĩa trong ngữ cảnh này cụ thể là gì?
        2. Có sắc thái văn hóa / slang gì không?
        Format JSON: {"pinyin":"...","meaning":"...","note":"...","register":"..."}
        Không giải thích dài dòng. Không thêm text ngoài JSON.
    """.trimIndent()
}
```

### SelectableText Component — kỹ thuật quan trọng

```kotlin
@Composable
fun SelectableChineseText(
    text: AnnotatedString,
    onTextSelected: (String, TextRange) -> Unit,
    modifier: Modifier = Modifier,
) {
    var selection by remember { mutableStateOf<TextRange?>(null) }
    val debounceScope = rememberCoroutineScope()
    var debounceJob by remember { mutableStateOf<Job?>(null) }

    BasicTextField(
        value = TextFieldValue(text, selection = selection ?: TextRange.Zero),
        onValueChange = { newValue ->
            selection = newValue.selection
            
            // Debounce 300ms — không trigger khi user vẫn đang kéo
            debounceJob?.cancel()
            debounceJob = debounceScope.launch {
                delay(300)
                val sel = newValue.selection
                if (!sel.collapsed && sel.length in 1..10) {
                    val selectedText = text.text.substring(sel.start, sel.end)
                    onTextSelected(selectedText, sel)
                }
            }
        },
        readOnly = true,
        modifier = modifier,
    )
}
```

---

## 4. Database Design

### PostgreSQL Schema (thực tế có thể build)

```sql
-- Users
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW(),
    topics      TEXT[] DEFAULT '{}'   -- ['humor', 'food', 'tech']
);

-- Content items (curated)
CREATE TABLE content_items (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    text         TEXT NOT NULL,
    source_type  VARCHAR(50) NOT NULL,  -- 'weibo', 'douyin_caption', 'news', 'dialogue'
    source_url   TEXT,
    topic        VARCHAR(100),
    difficulty   SMALLINT DEFAULT 3,    -- 1-5
    token_map    JSONB,                 -- [{start:0, end:2, word:"今天"}, ...]
    published_at TIMESTAMP,
    created_at   TIMESTAMP DEFAULT NOW()
);

-- Explanation cache (shared across users)
CREATE TABLE explain_cache (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    word         VARCHAR(100) NOT NULL,
    source_type  VARCHAR(50) NOT NULL,
    pinyin       VARCHAR(200),
    meaning      TEXT NOT NULL,
    cultural_note TEXT,
    register     VARCHAR(50),           -- 'internet_slang', 'formal', 'casual'
    model_used   VARCHAR(50),           -- 'gemini-gemma4-31b', 'qwen2.5-3b'
    created_at   TIMESTAMP DEFAULT NOW(),
    UNIQUE(word, source_type)
);

-- Memory capsules (per user)
CREATE TABLE memory_capsules (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id          UUID REFERENCES users(id) ON DELETE CASCADE,
    surface_form     VARCHAR(100) NOT NULL,  -- "甲方"
    context_window   TEXT NOT NULL,          -- 100 chars xung quanh
    source_content_id UUID REFERENCES content_items(id),
    explanation_json JSONB NOT NULL,         -- snapshot của explanation lúc save
    topics           TEXT[],
    saved_at         TIMESTAMP DEFAULT NOW()
);

-- FSRS learning state (per capsule per user)
CREATE TABLE learning_states (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capsule_id      UUID REFERENCES memory_capsules(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    stability       FLOAT DEFAULT 1.0,      -- FSRS: memory stability (days)
    difficulty      FLOAT DEFAULT 0.3,      -- FSRS: item difficulty
    due_date        TIMESTAMP NOT NULL,
    last_review     TIMESTAMP,
    review_count    INTEGER DEFAULT 0,
    state           VARCHAR(20) DEFAULT 'new', -- new/learning/review/relearning
    UNIQUE(capsule_id, user_id)
);

-- Indexes quan trọng
CREATE INDEX idx_capsules_user ON memory_capsules(user_id);
CREATE INDEX idx_learning_due ON learning_states(user_id, due_date);
CREATE INDEX idx_content_topic ON content_items(topic, published_at DESC);
CREATE INDEX idx_explain_cache ON explain_cache(word, source_type);
```

### Room Schema (Android — mirror của PostgreSQL)

```kotlin
@Database(
    entities = [
        ContentItemEntity::class,
        ExplainCacheEntity::class,
        MemoryCapsuleEntity::class,
        LearningStateEntity::class,
        SyncQueueEntity::class,   // Offline sync queue
    ],
    version = 1,
    exportSchema = true
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun contentDao(): ContentDao
    abstract fun explainCacheDao(): ExplainCacheDao
    abstract fun capsuleDao(): CapsuleDao
    abstract fun syncQueueDao(): SyncQueueDao
}
```

**Offline sync strategy:**  
Khi user không có mạng và save capsule → lưu vào `sync_queue` table.  
Khi mạng trở lại → WorkManager `SyncWorker` đẩy lên backend.  
Conflict resolution: last-write-wins dựa trên `saved_at` timestamp.

### pgvector (giai đoạn 2, không phải MVP)

```sql
-- Thêm extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Thêm column embedding vào explain_cache
ALTER TABLE explain_cache 
ADD COLUMN context_embedding vector(768);

-- Index cho similarity search
CREATE INDEX ON explain_cache 
USING ivfflat (context_embedding vector_cosine_ops)
WITH (lists = 100);

-- Query: "tìm 5 ngữ cảnh tương tự nhất khi dùng từ 厉害"
SELECT * FROM explain_cache
WHERE word = '厉害'
ORDER BY context_embedding <=> $1  -- $1 = embedding của context hiện tại
LIMIT 5;
```

---

## 5. Core Algorithms

### FSRS Algorithm (Kotlin thuần, không dependency)

FSRS (Free Spaced Repetition Scheduler) là thuật toán SRS hiện đại nhất, tốt hơn SM-2 của Anki về mặt toán học. Dưới đây là implementation đơn giản hóa (FSRS-4):

```kotlin
object FSRSAlgorithm {
    // Tham số mặc định (từ paper gốc)
    private val W = doubleArrayOf(
        0.4, 0.6, 2.4, 5.8, 4.93, 0.94, 0.86, 0.01,
        1.49, 0.14, 0.94, 2.18, 0.05, 0.34, 1.26, 0.29, 2.61
    )

    // rating: 1=Again, 2=Hard, 3=Good, 4=Easy
    fun nextState(current: LearningState, rating: Int): LearningState {
        val newDifficulty = nextDifficulty(current.difficulty, rating)
        val newStability = nextStability(current.stability, current.difficulty, rating, current.reviewCount)
        val interval = nextInterval(newStability)

        return current.copy(
            difficulty = newDifficulty,
            stability = newStability,
            dueDate = LocalDateTime.now().plusDays(interval.toLong()),
            reviewCount = current.reviewCount + 1,
            lastReview = LocalDateTime.now(),
        )
    }

    private fun nextDifficulty(d: Double, rating: Int): Double {
        val next = d - W[6] * (rating - 3)
        return next.coerceIn(1.0, 10.0)
    }

    private fun nextStability(s: Double, d: Double, rating: Int, n: Int): Double {
        return if (n == 0) {
            // First review
            W[rating - 1]
        } else {
            s * (Math.exp(W[8]) * (11 - d) * Math.pow(s, -W[9]) *
                (Math.exp((1 - W[10]) * 1.0) - 1) * (if (rating == 2) W[15] else if (rating == 4) W[16] else 1.0))
        }
    }

    private fun nextInterval(stability: Double): Int {
        val interval = (stability * 9.0).toInt()
        return interval.coerceIn(1, 365)
    }
}
```

### Context Recall Review (không phải flashcard)

```kotlin
// ReviewItem được tạo từ MemoryCapsule
data class ReviewItem(
    val capsuleId: UUID,
    val contextWindow: String,        // "今天被___气死了，真的是服了"
    val targetWord: String,           // "甲方" — bị ẩn đi
    val distractors: List<String>,    // 3 options sai được lấy từ capsules khác cùng topic
    val correctExplanation: String,   // Unlock sau khi trả lời đúng
)

// ReviewGenerator.kt
class ReviewGenerator @Inject constructor(
    private val capsuleDao: CapsuleDao
) {
    suspend fun generateReview(capsule: MemoryCapsule): ReviewItem {
        // Lấy 3 từ khác cùng topic làm distractor
        val distractors = capsuleDao
            .getByTopics(capsule.topics, exclude = capsule.id, limit = 3)
            .map { it.surfaceForm }

        // Mask từ target trong context
        val maskedContext = capsule.contextWindow
            .replace(capsule.surfaceForm, "___")

        return ReviewItem(
            capsuleId = capsule.id,
            contextWindow = maskedContext,
            targetWord = capsule.surfaceForm,
            distractors = distractors,
            correctExplanation = capsule.explanation,
        )
    }
}
```

### Content Recommendation (đơn giản nhưng hiệu quả)

```kotlin
// Không dùng ML phức tạp cho MVP
// Rule-based scoring:
fun scoreContent(item: ContentItem, userProfile: UserProfile): Double {
    var score = 0.0

    // 1. Topic match
    if (item.topic in userProfile.preferredTopics) score += 3.0

    // 2. Difficulty phù hợp với level user (ước tính từ review history)
    val diffDelta = abs(item.difficulty - userProfile.estimatedLevel)
    score += (3 - diffDelta).coerceAtLeast(0.0)

    // 3. Freshness (content mới hơn được ưu tiên)
    val daysSincePublished = ChronoUnit.DAYS.between(item.publishedAt, LocalDate.now())
    score += (7 - daysSincePublished).coerceAtLeast(0.0) * 0.5

    // 4. Không hiển thị lại content đã đọc
    if (item.id in userProfile.readContentIds) score = -999.0

    return score
}
```

---

## 6. AI / NLP — Chi tiết pipeline

### On-device: MediaPipe + Qwen2.5-3B

```kotlin
// OnDeviceEngine.kt
class OnDeviceEngine @Inject constructor(
    private val context: Context
) {
    private var inference: LlmInference? = null

    suspend fun initialize() = withContext(Dispatchers.IO) {
        val modelPath = context.filesDir.resolve("qwen2.5-3b-q4km.bin").absolutePath
        val options = LlmInference.LlmInferenceOptions.builder()
            .setModelPath(modelPath)
            .setMaxTokens(256)
            .setPreferredBackend(LlmInference.Backend.GPU) // tận dụng Adreno/Mali
            .build()
        inference = LlmInference.createFromOptions(context, options)
    }

    suspend fun explain(prompt: String): Explanation = withContext(Dispatchers.Default) {
        val raw = inference?.generateResponse(prompt) ?: return@withContext Explanation.fallback()
        parseJsonResponse(raw)
    }

    private fun parseJsonResponse(raw: String): Explanation {
        return try {
            val json = raw.extractJson() // tìm {...} trong output
            val obj = JSONObject(json)
            Explanation(
                pinyin = obj.optString("pinyin"),
                meaning = obj.optString("meaning"),
                culturalNote = obj.optString("note"),
                register = obj.optString("register"),
                source = ExplanationSource.ON_DEVICE,
            )
        } catch (e: Exception) {
            // Nếu model không ra JSON đúng format → parse heuristic
            Explanation(meaning = raw.take(200), source = ExplanationSource.ON_DEVICE)
        }
    }
}
```

### Cloud: Gemini API (gọi trực tiếp từ app, không qua backend)

```kotlin
// CloudEngine.kt
class CloudEngine @Inject constructor(
    private val userPreferences: UserPreferences,
    private val okHttpClient: OkHttpClient,
) {
    suspend fun explain(word: String, context: String, sourceType: String): Explanation {
        val apiKey = userPreferences.geminiApiKey ?: throw NoApiKeyException()
        
        val requestBody = buildGeminiRequest(word, context, sourceType)
        
        val request = Request.Builder()
            .url("https://generativelanguage.googleapis.com/v1beta/models/gemma-4-31b-it:generateContent?key=$apiKey")
            .post(requestBody)
            .build()
        
        val response = okHttpClient.newCall(request).await() // coroutine extension
        return parseGeminiResponse(response)
    }

    private fun buildGeminiRequest(word: String, context: String, sourceType: String): RequestBody {
        val systemPrompt = """
            Bạn là trợ lý giải nghĩa tiếng Trung cho người Việt.
            Chỉ trả về JSON, không text thêm.
            Schema: {"pinyin":"string","meaning":"string","note":"string|null","register":"formal|casual|internet_slang|spoken"}
            
            QUAN TRỌNG: Giải thích nghĩa TRONG NGỮ CẢNH này, không phải nghĩa từ điển.
            Nếu là slang/meme → note phải giải thích cultural context.
            Source type: $sourceType
        """.trimIndent()
        
        val json = JSONObject().apply {
            put("contents", JSONArray().apply {
                put(JSONObject().apply {
                    put("role", "user")
                    put("parts", JSONArray().apply {
                        put(JSONObject().put("text", 
                            "$systemPrompt\n\nTừ: \"$word\"\nNgữ cảnh: \"$context\""))
                    })
                })
            })
            put("generationConfig", JSONObject().apply {
                put("maxOutputTokens", 300)
                put("temperature", 0.2) // thấp = consistent, ít sáng tạo
                put("topP", 0.8)
            })
        }
        
        return json.toString().toRequestBody("application/json".toMediaType())
    }
}
```

### Python NLP Sidecar (nhỏ, chạy cùng server)

```python
# nlp_sidecar/main.py — FastAPI, deploy cùng Docker Compose
from fastapi import FastAPI
import jieba
import jieba.posseg as pseg
from pypinyin import lazy_pinyin

app = FastAPI()

@app.post("/tokenize")
def tokenize(text: str):
    words = pseg.cut(text)
    result = []
    offset = 0
    for word, pos in words:
        result.append({
            "word": word,
            "pos": pos,           # n=noun, v=verb, etc.
            "start": offset,
            "end": offset + len(word),
            "pinyin": lazy_pinyin(word),
        })
        offset += len(word)
    return {"tokens": result}

@app.post("/hard-words")
def get_hard_words(text: str, n: int = 20):
    """Lấy n từ khó nhất (không có trong top 3000 từ phổ thông)"""
    tokens = [w for w, _ in pseg.cut(text) if len(w) > 1]
    hard = [t for t in tokens if t not in COMMON_WORDS_3000]
    return {"words": hard[:n]}
```

### Khi nào rule-based, khi nào AI?

| Task | Approach | Lý do |
|---|---|---|
| Tách từ (segmentation) | Rule-based (jieba) | Deterministic, nhanh, proven |
| Pinyin | Rule-based (pypinyin) | 100% chính xác |
| Phát hiện từ khó | Rule-based (so với wordlist) | Không cần AI |
| Contextual explanation | AI (LLM) | Rule không capture được cultural layer |
| SRS scheduling | Algorithm (FSRS) | Math thuần túy |
| Content recommendation | Weighted scoring | Đủ tốt cho MVP, không cần ML |
| Register detection | AI (từ explanation response) | LLM tự classify trong JSON |

### Giảm cost + latency — cụ thể

**Cache strategy:**
```
Cache key = SHA256(word + "::" + source_type)
TTL = 7 ngày (explanation không đổi)
Hit rate estimate: ~80% sau tuần đầu (user hay highlight từ phổ biến)
```

**Cost estimate thực tế:**  
- Gemma 4 31B trên Google AI Studio: ~$0.003/1K input tokens  
- Một explanation request ≈ 200 input tokens  
- = $0.0006/request  
- 1000 request/ngày không cache = $0.60/ngày  
- Với 80% cache hit rate = $0.12/ngày — rất hợp lý

**Rate limit per user (free tier):**  
10 cloud requests/ngày → fallback to on-device sau đó  
Store count trong Redis: `INCR user:{id}:explain:count` với TTL 24h

---

## 7. Hướng dẫn triển khai — Step by step

### Phase 1: MVP (Tuần 1-4)

**Mục tiêu:** App chạy được, core flow hoạt động end-to-end

**Tuần 1 — Foundation:**
- [ ] Setup Android project (Kotlin, Compose, Hilt, Room, Retrofit)
- [ ] Setup Spring Boot project (Spring Web, Spring Data JPA, PostgreSQL)
- [ ] Docker Compose cho local dev (PostgreSQL + Redis + NLP sidecar)
- [ ] Basic auth (JWT) — register/login
- [ ] Seed 20 content items thủ công vào DB

**Tuần 2 — Core Android:**
- [ ] ContentFeed screen (list view, đọc từ backend)
- [ ] ReadingScreen với SelectableChineseText
- [ ] Tích hợp on-device model (MediaPipe + Qwen2.5-3B download)
- [ ] ExplanationBubble UI component
- [ ] ExplainWordUseCase (on-device only trước, cloud sau)

**Tuần 3 — Core Backend + Cloud AI:**
- [ ] Explain cache API (GET/POST)
- [ ] Gemini cloud integration trong app
- [ ] Hybrid flow: on-device fast → cloud enhanced
- [ ] Save capsule (local Room DB)
- [ ] Sync capsules lên backend

**Tuần 4 — Review system + Polish:**
- [ ] FSRS algorithm implementation
- [ ] ReviewScreen (Context Recall UX)
- [ ] Push notification cho review reminders
- [ ] Settings screen (API key input, model download UI)
- [ ] Offline fallback graceful handling

**Definition of MVP done:**
- User có thể đọc content → highlight từ → nhận explanation trong 2s
- User có thể save capsule
- User nhận notification ôn tập và hoàn thành review
- Chạy được offline (degraded mode)

---

### Phase 2: V1 (Tuần 5-8)

- [ ] Python NLP sidecar deployed, CacheWarmup job chạy
- [ ] ContentIngestion pipeline (admin có thể thêm content dễ dàng)
- [ ] Topic selection onboarding
- [ ] Content feed recommendation (rule-based scoring)
- [ ] Learning progress view ("bản đồ chủ đề")
- [ ] Multi-source content (không chỉ text — transcript từ video)
- [ ] A/B test on-device vs cloud explanation quality

---

### Phase 3: Scale (sau V1)

- [ ] pgvector integration (context similarity search)
- [ ] Fine-tune Qwen2.5-3B với accumulated explanation data (LoRA)
- [ ] On-device model tự cập nhật qua delta patches
- [ ] Collaborative filtering: "user tương tự thường lưu từ nào"
- [ ] Analytics dashboard cho admin (từ nào được highlight nhiều nhất)
- [ ] iOS port (KMM cho business logic, SwiftUI cho UI)

---

## 8. Tech Stack Cụ thể

### Android
```
Language:       Kotlin 2.0
UI:             Jetpack Compose + Material 3
Architecture:   Clean Architecture + MVI
DI:             Hilt (Dagger)
Async:          Coroutines + Flow
Local DB:       Room 2.6
Network:        Retrofit 2 + OkHttp 4
Image:          Coil 2
On-device AI:   MediaPipe LLM Inference API 0.10+
Prefs:          DataStore (EncryptedSharedPreferences cho API key)
Background:     WorkManager
Testing:        JUnit 5 + MockK + Turbine (Flow testing)
```

### Backend
```
Language:       Java 21 (Virtual Threads — Project Loom)
Framework:      Spring Boot 3.3
ORM:            Spring Data JPA + Hibernate
Migration:      Flyway
Auth:           Spring Security + JWT (jjwt)
Cache:          Spring Cache + Redis (Lettuce)
Job:            Spring @Scheduled
HTTP Client:    WebClient (cho Gemini calls từ server)
Build:          Gradle (Kotlin DSL)
Testing:        JUnit 5 + Testcontainers
```

**Tại sao Java 21 + Virtual Threads?**  
Khi có nhiều concurrent Gemini API calls (mỗi call block ~5s), Virtual Threads cho phép handle hàng nghìn concurrent calls mà không cần reactive programming. Code đơn giản hơn WebFlux rất nhiều — phù hợp 1 dev.

### Database
```
Primary:        PostgreSQL 16
Caching:        Redis 7 (explain cache, rate limiting)
Search:         pgvector (phase 2)
Migrations:     Flyway
```

### NLP Sidecar
```
Language:       Python 3.11
Framework:      FastAPI
Libraries:      jieba, pypinyin, opencc-python-reimplemented
Deploy:         Docker container cùng Compose
```

### Infrastructure
```
Deploy:         Railway.app (đơn giản nhất cho 1 dev)
                hoặc Render.com
                (Sau này → AWS ECS khi cần scale)
Container:      Docker + Docker Compose
CDN/Storage:    Cloudflare R2 (lưu model file để user download)
Monitoring:     Sentry (error tracking) + Grafana Cloud (free tier)
CI/CD:          GitHub Actions
```

**Tại sao Railway thay vì AWS ngay từ đầu?**  
Railway deploy từ Git push, setup PostgreSQL + Redis trong 5 phút, free tier đủ cho MVP. Khi cần scale → migrate sang AWS ECS/RDS. Đừng over-engineer infra khi chưa có user.

### CI/CD (GitHub Actions)
```yaml
# .github/workflows/backend.yml
on: [push]
jobs:
  test-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21' }
      - run: ./gradlew test
      - run: ./gradlew bootJar
      - uses: railwayapp/railway-deploy-action@v1
        with: { service: backend }
```

---

## 9. Trade-offs & Risks

### Rủi ro 1: On-device model không chạy được trên máy yếu
**Xác suất:** Cao  
**Impact:** Trung bình  
**Giảm thiểu:**
- Detect RAM lúc runtime: nếu < 4GB → skip model download, chỉ dùng dictionary offline
- Offer "Cloud-only mode" cho máy yếu
- Quantize aggressively hơn (Q3_K_S) — smaller nhưng kém hơn chút

### Rủi ro 2: Gemini API key management phức tạp cho user
**Xác suất:** Cao  
**Impact:** Cao (friction cản user onboarding)  
**Giảm thiểu:**
- Làm onboarding screen cực kỳ đơn giản với link trực tiếp đến Google AI Studio
- Video hướng dẫn 60 giây ngay trong app
- Free tier: app có sẵn server-side API key giới hạn 5 requests/ngày/user — đủ để trải nghiệm trước khi user quyết định thêm key riêng

### Rủi ro 3: Content curation không scale
**Xácsuất:** Cao (long-term)  
**Impact:** Trung bình  
**Giảm thiểu:**
- Tuần 1-8: founder tự curate, đủ cho 200-300 items
- Phase 2: semi-automated pipeline (RSS feed từ 澎湃新闻 + keyword filter)
- Phase 3: Community submission với moderation queue

### Rủi ro 4: LLM output không consistent
**Xác suất:** Trung bình  
**Impact:** Trung bình  
**Giảm thiểu:**
- Temperature thấp (0.2) để giảm randomness
- Validate JSON schema trước khi show user
- Fallback parsing heuristic nếu JSON malformed
- Log rate của malformed responses → nếu > 5% thì tune prompt

### Bottleneck có thể gặp khi scale

| Bottleneck | Xảy ra khi | Giải pháp |
|---|---|---|
| PostgreSQL slow | >10k MAU | Read replicas + connection pooling (PgBouncer) |
| Redis cache miss | Content mới chưa warm | Increase warmup window từ 2h lên 6h |
| Gemini API rate limit | >100 req/min | Implement queue + exponential backoff |
| On-device inference OOM | Foreground + background app | Release model khi app background >5 phút |

---

## 10. Developer Execution Plan (1 dev, 4 tuần)

### Nguyên tắc làm việc

**Không over-engineer** = Đừng build gì mà tuần 1 chưa cần.  
**Vertical slices** = Mỗi ngày ship một feature nhỏ chạy được end-to-end.  
**Fake it till you make it** = Content đầu tiên hardcode cũng được. Auth đầu tiên có thể skip.

---

### Tuần 1: Foundation (Backend + Android shell)

| Ngày | Task | Output |
|---|---|---|
| T2 | Setup Android project, Hilt, Room, Compose navigation | App chạy được, blank screens |
| T3 | Setup Spring Boot, Docker Compose, PostgreSQL | Backend chạy local |
| T4 | Seed 10 content items vào DB, Content API + Android feed screen | Đọc được danh sách content |
| T5 | Auth backend (JWT) + Android login screen | Login hoạt động |
| T6 | ReadingScreen + SelectableChineseText component | Chọn được text |

**Bỏ qua tuần 1:** Cloud AI, SRS, notifications, proper error handling.

---

### Tuần 2: On-device AI (khó nhất về kỹ thuật)

| Ngày | Task | Output |
|---|---|---|
| T2 | Download Qwen2.5-3B Q4KM, setup MediaPipe | Model load thành công |
| T3 | OnDeviceEngine.explain() với test prompt | JSON output từ model |
| T4 | ExplainWordUseCase (on-device only) + bubble UI | Highlight → thấy explanation |
| T5 | Tune prompt cho on-device (nhiều iteration) | Output chất lượng tốt |
| T6 | Error handling + loading states + animations | UX mượt |

**Lưu ý tuần 2:** Prompt tuning mất nhiều thời gian nhất. Reserve cả ngày T5.

---

### Tuần 3: Cloud AI + Save/Sync

| Ngày | Task | Output |
|---|---|---|
| T2 | CloudEngine.kt (Gemini API call) | Cloud explanation hoạt động |
| T3 | Hybrid flow (on-device fast + cloud update) | Badge Quick/Enhanced |
| T4 | Save capsule (Room local) | User có thể save |
| T5 | Backend capsule API + Android sync | Sync online |
| T6 | Explain cache API (server-side) | Cache hit từ server |

---

### Tuần 4: Review + Polish

| Ngày | Task | Output |
|---|---|---|
| T2 | FSRS algorithm + LearningState | Scheduling hoạt động |
| T3 | ReviewScreen UX (Context Recall) | User có thể ôn tập |
| T4 | WorkManager notifications | Nhận reminder |
| T5 | Settings screen (API key + model download flow) | Onboarding hoàn chỉnh |
| T6 | Bug fix + offline testing + demo build | MVP ready |

---

### Checklist trước khi ship MVP

- [ ] App không crash khi offline
- [ ] Explanation xuất hiện trong <2s (on-device path)
- [ ] User có thể complete full loop: read → highlight → explain → save → review
- [ ] API key được lưu secure (EncryptedSharedPreferences)
- [ ] Không leak memory khi inference (model release đúng chỗ)
- [ ] Backend deployed trên Railway
- [ ] 20+ content items để test

---

## Appendix: Folder structure đầy đủ để bắt đầu code

```bash
# Clone template và bắt đầu
git clone <repo>

chenjin/
├── android/                    # Android app
├── backend/                    # Spring Boot
├── nlp-sidecar/               # Python FastAPI
├── docker-compose.yml          # Local dev
├── .github/workflows/          # CI/CD
└── docs/
    ├── api.md                  # API documentation
    └── content-guide.md        # Hướng dẫn thêm content
```

```yaml
# docker-compose.yml — chạy ngay được
version: '3.9'
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: chenjin
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpass
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  nlp-sidecar:
    build: ./nlp-sidecar
    ports: ["8001:8001"]
    environment:
      - PYTHONUNBUFFERED=1

  backend:
    build: ./backend
    ports: ["8080:8080"]
    depends_on: [postgres, redis, nlp-sidecar]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/chenjin
      SPRING_DATASOURCE_USERNAME: dev
      SPRING_DATASOURCE_PASSWORD: devpass
      SPRING_REDIS_HOST: redis
      NLP_SIDECAR_URL: http://nlp-sidecar:8001

volumes:
  pgdata:
```

Chạy `docker compose up` → backend sẵn sàng tại `localhost:8080`.  
Android trỏ đến `10.0.2.2:8080` (emulator) hoặc IP máy thật.

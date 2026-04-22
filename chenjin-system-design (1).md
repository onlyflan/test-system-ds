# 沉浸 (Chénjìn) — Technical Blueprint
> Senior Architect Edition · Build-ready · 1 dev / small team  
> *v2 — cập nhật: thay on-device LLM bằng SQLite dictionary (lý do ở mục 6)*

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
- Explanation phải xuất hiện trong ≤50ms (dictionary path) hoặc ≤8s (cloud path cho từ mới)
- Hoạt động offline hoàn toàn với dictionary bundled trong APK
- Chi phí LLM kiểm soát được qua server-side cache warmup

---

## 1. System Design — High Level

```
┌─────────────────────────────────────────────────────────────────┐
│                        ANDROID APP                              │
│                                                                 │
│  ContentFeed ──► ReadingScreen ──► HighlightGesture             │
│                        │                                        │
│            ┌───────────┴─────────────┐                         │
│            ▼                         ▼                          │
│   LocalDictionary (~30MB SQLite)   CloudEngine (async)          │
│   bundled trong APK, instant       Gemini API, user key         │
│            │                         │                          │
│    <50ms lookup                  ~3-8s response                 │
│    offline 100%                  cần mạng + API key             │
│            │                         │                          │
│            └──────────┬──────────────┘                         │
│                       ▼                                         │
│               ExplainCacheDao (Room DB)                         │
│               lưu cloud results để reuse                        │
│                       │                                         │
│                 SaveCapsule?                                     │
│                       ▼                                         │
│             MemoryCapsuleStore (Room DB)                        │
│                       │                                         │
│             ReviewScheduler (FSRS algo)                         │
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
2. Android: debounce 300ms → extract (word, context 100 chars xung quanh)
3. Android: check Room cache → miss
4. Android: check LocalDictionary (SQLite lookup, <50ms)
   → HIT (80-85% từ phổ thông):
     └─ Emit Quick ngay lập tức
     └─ Cloud chạy background để enhance
   → MISS (15-20% từ khó/slang mới):
     └─ Emit Loading
     └─ Chờ cloud (~3-8s)
5. Cloud response về:
   → Lưu vào Room cache
   → Emit Enhanced → UI update in-place (badge đổi)
6. User nhấn ⊕ Save → tạo MemoryCapsule → sync lên backend khi online
7. Backend: FSRS tính due_date → schedule review notification
```

**Tại sao dictionary thay vì on-device LLM:**  
Qwen2.5-3B Q4KM thực tế mất 8-15s ngay cả với GPU delegation. Model nhỏ hơn (1.5B) cho ~3-5s — vẫn đủ để user bỏ highlight. Model đủ nhỏ để <2s (0.5B) thì không hiểu được slang và cultural nuance. Dictionary lookup <50ms giải quyết 80% trường hợp; 20% còn lại cần cloud quality — đúng là không phải model nhỏ.

---

## 2. Kiến trúc chi tiết — Backend

### Module structure (Spring Boot)

```
chenjin-backend/
├── src/main/java/com/chenjin/
│   ├── content/
│   │   ├── ContentController.java
│   │   ├── ContentService.java
│   │   ├── ContentRepository.java
│   │   └── domain/
│   │       ├── ContentItem.java
│   │       └── TokenMap.java
│   │
│   ├── explain/
│   │   ├── ExplainCacheController.java
│   │   ├── ExplainCacheService.java
│   │   ├── ExplainCacheRepository.java
│   │   └── domain/ExplainCache.java
│   │
│   ├── capsule/
│   │   ├── CapsuleController.java
│   │   ├── CapsuleService.java
│   │   ├── CapsuleRepository.java
│   │   └── domain/
│   │       ├── MemoryCapsule.java
│   │       └── LearningState.java
│   │
│   ├── user/
│   │   ├── AuthController.java
│   │   ├── UserService.java
│   │   └── domain/User.java
│   │
│   ├── jobs/
│   │   ├── CacheWarmupJob.java
│   │   └── ReviewReminderJob.java
│   │
│   └── shared/
│       ├── srs/FSRSService.java
│       ├── config/SecurityConfig.java
│       └── config/RedisConfig.java
│
└── src/main/resources/application.yml
```

### API Design

```yaml
# Content
GET  /api/v1/feed?topic=humor&offset=0&limit=10
GET  /api/v1/content/{id}

# Explain cache — client check trước khi dùng user API key
GET  /api/v1/explain/cache?word=甲方&source_type=weibo
     → 200: cache hit (user không tốn quota)
     → 404: miss (client gọi Gemini với user key)

# Capsules
POST /api/v1/capsules
GET  /api/v1/capsules/due
PUT  /api/v1/capsules/{id}/review

# Auth
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/refresh
```

### CacheWarmup Job — quan trọng nhất về performance

```java
@Component
public class CacheWarmupJob {

    @Scheduled(fixedDelayString = "${content.warmup-interval-hours}h",
               initialDelayString = "PT30S")
    public void warmUpNewContent() {
        // Content scheduled push trong 2h tới
        List<ContentItem> upcoming = getUpcomingContent();

        for (ContentItem item : upcoming) {
            // NLP sidecar: tokenize + lọc từ khó
            List<String> hardWords = nlpClient.getHardWords(item.getText(), 20);

            for (String word : hardWords) {
                if (explainCacheService.findCached(word, item.getSourceType()) != null)
                    continue; // đã có, skip

                String context = extractContextWindow(item.getText(), word);
                ExplainResponse exp = geminiClient.explain(word, context, item.getSourceType());
                explainCacheService.store(word, item.getSourceType(), exp);

                Thread.sleep(200); // rate limiting ~300 req/min max
            }
        }
    }
}
```

Khi job này hoạt động đúng: user highlight từ bất kỳ trong content mới → server cache có sẵn → client nhận ~100ms thay vì 3-8s.

---

## 3. Kiến trúc chi tiết — Android

### Clean Architecture + MVI

```
app/
├── data/
│   ├── local/
│   │   ├── db/AppDatabase.kt
│   │   ├── dao/CapsuleDao.kt
│   │   ├── dao/ExplainCacheDao.kt
│   │   ├── dao/ContentDao.kt
│   │   └── prefs/UserPreferences.kt
│   ├── remote/
│   │   ├── api/BackendApi.kt
│   │   └── dto/
│   └── repository/
│
├── domain/
│   ├── model/
│   ├── usecase/
│   │   ├── ExplainWordUseCase.kt    # dictionary-first hybrid flow
│   │   ├── SaveCapsuleUseCase.kt
│   │   ├── GetFeedUseCase.kt
│   │   └── GetDueReviewsUseCase.kt
│   └── srs/FSRSAlgorithm.kt
│
├── presentation/
│   ├── feed/
│   ├── reading/
│   │   ├── ReadingViewModel.kt
│   │   ├── ReadingScreen.kt
│   │   └── components/
│   │       ├── SelectableChineseText.kt
│   │       └── ExplanationBubble.kt
│   ├── review/
│   └── settings/
│       └── SettingsScreen.kt        # API key setup (không cần model download)
│
└── ai/
    ├── ondevice/
    │   └── LocalDictionaryEngine.kt # SQLite lookup, <50ms, offline
    └── cloud/
        └── CloudEngine.kt           # Gemini API, context-aware
```

### State management — MVI pattern

```kotlin
data class ReadingState(
    val content: ContentItem? = null,
    val isContentLoading: Boolean = true,
    val selectedText: String = "",
    val selectionRange: TextRange? = null,
    val explanationState: ExplanationUiState = ExplanationUiState.Idle,
    val snackbarMessage: String? = null,
)

sealed class ExplanationUiState {
    object Idle : ExplanationUiState()
    object Loading : ExplanationUiState()
    // Quick = dict hit, cloud đang chạy background (isUpgrading=true)
    data class QuickResult(val explanation: Explanation, val isUpgrading: Boolean = true)
        : ExplanationUiState()
    // Enhanced = cloud result hoặc Room cache
    data class EnhancedResult(val explanation: Explanation) : ExplanationUiState()
    data class Error(val message: String) : ExplanationUiState()
}

sealed class ReadingIntent {
    data class TextSelected(val text: String, val range: TextRange) : ReadingIntent()
    object DismissExplanation : ReadingIntent()
    data class SaveCurrentExplanation(val explanation: Explanation) : ReadingIntent()
}
```

### ExplainWordUseCase — dictionary-first flow

```kotlin
class ExplainWordUseCase @Inject constructor(
    private val dictEngine: LocalDictionaryEngine,
    private val cloudEngine: CloudEngine,
    private val explainCacheDao: ExplainCacheDao,
) {
    fun explain(word: String, contextWindow: String, sourceType: String)
        : Flow<ExplanationResult> = flow {

        // 1. Room cache — cloud-quality, persistent
        val cached = explainCacheDao.get(word, sourceType)
        if (cached != null) {
            emit(ExplanationResult.Enhanced(cached.toExplanation()))
            return@flow
        }

        // 2. Local dictionary — instant, offline-first
        val dictResult = runCatching { dictEngine.lookup(word, sourceType) }.getOrNull()
        if (dictResult != null) {
            emit(ExplanationResult.Quick(dictResult))
            // cloud vẫn chạy để enhance + populate cache
        } else {
            emit(ExplanationResult.Loading)
            // user thấy "đang phân tích..." trong lúc cloud xử lý
        }

        // 3. Cloud — luôn chạy
        runCatching { cloudEngine.explain(word, contextWindow, sourceType) }
            .onSuccess { explanation ->
                explainCacheDao.insert(explanation.toEntity(word, sourceType))
                emit(ExplanationResult.Enhanced(explanation))
            }
            .onFailure { error ->
                if (dictResult == null) {
                    emit(ExplanationResult.Error(
                        if ("API key" in (error.message ?: ""))
                            "Chưa thiết lập Gemini API key trong cài đặt."
                        else "Không thể giải thích. Kiểm tra kết nối mạng."
                    ))
                }
                // Dict đã có result → im lặng
            }
    }
}
```

### SelectableChineseText — kỹ thuật quan trọng

```kotlin
@Composable
fun SelectableChineseText(
    text: String,
    onTextSelected: (String, TextRange) -> Unit,
    modifier: Modifier = Modifier,
) {
    val scope = rememberCoroutineScope()
    var debounceJob by remember { mutableStateOf<Job?>(null) }
    var tfv by remember { mutableStateOf(TextFieldValue(text)) }

    BasicTextField(
        value = tfv,
        onValueChange = { newValue ->
            tfv = newValue
            val sel = newValue.selection
            debounceJob?.cancel()
            debounceJob = scope.launch {
                delay(300)
                if (!sel.collapsed && sel.length in 1..8) {
                    val selected = text.substring(
                        sel.start.coerceIn(0, text.length),
                        sel.end.coerceIn(0, text.length)
                    )
                    if (selected.any { it.code in 0x4E00..0x9FFF })
                        onTextSelected(selected, sel)
                }
            }
        },
        readOnly = true,
        textStyle = MaterialTheme.typography.bodyLarge.copy(lineHeight = 32.sp),
        modifier = modifier,
    )
}
```

---

## 4. Database Design

### PostgreSQL Schema

```sql
CREATE TABLE users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email        VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    topics       TEXT[] DEFAULT '{}',
    created_at   TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE content_items (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    text          TEXT NOT NULL,
    source_type   VARCHAR(50) NOT NULL,
    source_url    TEXT,
    topic         VARCHAR(100) NOT NULL,
    difficulty    SMALLINT DEFAULT 3,
    token_map     JSONB DEFAULT '[]',
    published_at  TIMESTAMP,
    scheduled_for TIMESTAMP,
    is_active     BOOLEAN DEFAULT TRUE
);

-- Shared cache — model_used: 'gemini-server' (warmup) atau 'gemini-user'
CREATE TABLE explain_cache (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    word          VARCHAR(200) NOT NULL,
    source_type   VARCHAR(50) NOT NULL,
    pinyin        VARCHAR(200),
    meaning       TEXT NOT NULL,
    cultural_note TEXT,
    register      VARCHAR(50) NOT NULL,
    example       TEXT,
    model_used    VARCHAR(100) NOT NULL,
    hit_count     INTEGER DEFAULT 0,
    created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(word, source_type)
);

CREATE TABLE memory_capsules (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id           UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    surface_form      VARCHAR(200) NOT NULL,
    context_window    TEXT NOT NULL,
    source_content_id UUID REFERENCES content_items(id) ON DELETE SET NULL,
    explanation       JSONB NOT NULL,
    topics            TEXT[] DEFAULT '{}',
    saved_at          TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE learning_states (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    capsule_id   UUID NOT NULL REFERENCES memory_capsules(id) ON DELETE CASCADE,
    user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    stability    FLOAT NOT NULL DEFAULT 1.0,
    difficulty   FLOAT NOT NULL DEFAULT 0.3,
    due_date     TIMESTAMP NOT NULL DEFAULT NOW() + INTERVAL '1 day',
    last_review  TIMESTAMP,
    review_count INTEGER NOT NULL DEFAULT 0,
    state        VARCHAR(20) NOT NULL DEFAULT 'NEW',
    UNIQUE(capsule_id, user_id)
);

CREATE INDEX idx_content_scheduled ON content_items(scheduled_for) WHERE is_active;
CREATE INDEX idx_explain_cache_word ON explain_cache(word, source_type);
CREATE INDEX idx_capsules_user ON memory_capsules(user_id);
CREATE INDEX idx_learning_due ON learning_states(user_id, due_date);
```

### Room Schema (Android)

```kotlin
@Database(
    entities = [
        ContentItemEntity::class,
        ExplainCacheEntity::class,   // cloud results, persistent
        MemoryCapsuleEntity::class,
        LearningStateEntity::class,
        SyncQueueEntity::class,      // offline sync queue
    ],
    version = 1,
    exportSchema = true,
)
abstract class AppDatabase : RoomDatabase() { ... }
```

**LocalDictionary là file riêng, không phải Room:**  
`assets/chenjin_dict.db` (~30MB) mở bằng `SQLiteDatabase.openDatabase()` read-only.  
Không dùng Room vì dict không cần ORM, migrate, hay write — chỉ cần lookup nhanh.

**Offline sync:** Save capsule offline → `sync_queue`. WorkManager `SyncWorker` flush khi có mạng. Conflict: last-write-wins theo `saved_at`.

### pgvector (Phase 2)

```sql
CREATE EXTENSION IF NOT EXISTS vector;
ALTER TABLE explain_cache ADD COLUMN context_embedding vector(768);
CREATE INDEX ON explain_cache USING ivfflat (context_embedding vector_cosine_ops);

-- "Tìm lần 厉害 được dùng tương tự context này"
SELECT * FROM explain_cache
WHERE word = '厉害'
ORDER BY context_embedding <=> $1
LIMIT 5;
```

---

## 5. Core Algorithms

### FSRS-4 (Kotlin thuần, zero dependency)

```kotlin
object FSRSAlgorithm {
    private val W = doubleArrayOf(
        0.4072, 1.1829, 3.1262, 15.4722, 7.2102, 0.5316, 1.0651, 0.0589,
        1.5330, 0.1544, 0.9724, 1.9813, 0.0953, 0.2975, 2.2042, 0.2407, 2.9466
    )

    fun nextState(current: LearningState, rating: ReviewRating): LearningState {
        val r = rating.value
        val newDiff = nextDifficulty(current.difficulty, r)
        val newStab = when {
            current.reviewCount == 0 -> W[r - 1]
            r == 1 -> nextForgetStability(newDiff, current.stability)
            else   -> nextRecallStability(newDiff, current.stability, r)
        }
        val interval = (newStab / 0.9 * (0.9.pow(1.0 / -0.5) - 1))
            .toInt().coerceIn(1, 365)
        return current.copy(
            difficulty = newDiff,
            stability  = newStab,
            dueDate    = LocalDateTime.now().plusDays(interval.toLong()),
            reviewCount = current.reviewCount + 1,
            lastReview  = LocalDateTime.now(),
            state = if (r == 1) SRSState.RELEARNING else SRSState.REVIEW,
        )
    }

    private fun nextDifficulty(d: Double, r: Int) =
        (d - W[6] * (r - 3) + W[7] * (W[4] - d)).coerceIn(1.0, 10.0)

    private fun nextRecallStability(d: Double, s: Double, r: Int) =
        s * (exp(W[8]) * (11 - d) * s.pow(-W[9]) *
             (exp(0.9 * W[10]) - 1) *
             (if (r == 2) W[15] else 1.0) *
             (if (r == 4) W[16] else 1.0)) + s

    private fun nextForgetStability(d: Double, s: Double) =
        W[11] * d.pow(-W[12]) * ((s + 1).pow(W[13]) - 1) * exp(0.1 * W[14])
}
```

### Context Recall Review

```kotlin
// Tạo review item từ capsule: mask từ target, lấy 3 distractors cùng topic
suspend fun generateReview(capsule: MemoryCapsule): ReviewItem {
    val masked = capsule.contextWindow.replace(capsule.surfaceForm, "___")
    val distractors = capsuleDao
        .getByTopic(topic = capsule.topics.firstOrNull() ?: "general",
                    excludeId = capsule.id, limit = 3)
        .map { it.surfaceForm }
    val options = (distractors + capsule.surfaceForm).shuffled()
    return ReviewItem(capsule.id, masked, options,
                      correctIndex = options.indexOf(capsule.surfaceForm),
                      explanation = capsule.explanation)
}
```

### Content Recommendation (rule-based, đủ cho MVP)

```kotlin
fun scoreContent(item: ContentItem, profile: UserProfile): Double {
    var score = 0.0
    if (item.topic in profile.preferredTopics) score += 3.0
    score += (3 - abs(item.difficulty - profile.estimatedLevel)).coerceAtLeast(0.0)
    val days = ChronoUnit.DAYS.between(item.publishedAt, LocalDate.now())
    score += (7 - days).coerceAtLeast(0.0) * 0.5
    if (item.id in profile.readContentIds) score = -999.0
    return score
}
```

---

## 6. AI / NLP — Chi tiết pipeline

### Tại sao bỏ on-device LLM (quyết định thực tế)

On-device LLM ban đầu được thiết kế để cung cấp "instant response trong khi cloud xử lý". Sau khi test thực tế:

| Model | Kết quả thực tế |
|---|---|
| Qwen2.5-3B Q4KM (~1.9GB) | 8-15s kể cả GPU delegation — quá chậm cho UX |
| Qwen2.5-1.5B Q4KM (~950MB) | 3-5s — vẫn đủ để user bỏ highlight |
| Qwen2.5-0.5B Q4KM (~350MB) | ~1-2s — đủ nhanh nhưng không hiểu slang, cultural nuance |

Không có điểm giao giữa "đủ nhanh" và "đủ chất lượng" trên Android hiện tại. SQLite dictionary giải quyết bài toán instant response tốt hơn với chi phí 0ms latency, không có ceiling về accuracy (curated data), và không yêu cầu 1-2GB RAM.

### LocalDictionary — SQLite bundled trong APK

```
assets/chenjin_dict.db   (~25-30MB)
assets/dict_version.txt  ("1")
```

```sql
CREATE TABLE dictionary (
    word          TEXT NOT NULL UNIQUE,
    pinyin        TEXT,
    meaning_vi    TEXT NOT NULL,
    cultural_note TEXT,
    register      TEXT DEFAULT 'casual',
    example       TEXT
);
CREATE INDEX idx_word ON dictionary(word);
```

Tạo dictionary một lần bằng `build_dictionary.py`:
```bash
# Gemini Flash cho 15k từ: ~$0.25, ~2-3 giờ
python nlp-sidecar/build_dictionary.py \
  --api-key AIza... \
  --word-list nlp-sidecar/words.txt \
  --output chenjin_dict.db

cp chenjin_dict.db android/app/src/main/assets/
echo "1" > android/app/src/main/assets/dict_version.txt
```

Dictionary update: tăng version → app detect và copy file mới khi APK update.

### Cloud: Gemini API (user key, gọi trực tiếp từ app)

```kotlin
// System prompt cho cloud — context-aware, quality cao
val systemPrompt = """
    Bạn là chuyên gia giải nghĩa tiếng Trung hiện đại cho người Việt học.
    Trả về ĐÚNG JSON, không text thêm:
    {"pinyin":"...","meaning":"...","note":null,"register":"...","example":"..."}

    QUAN TRỌNG: giải thích nghĩa TRONG NGỮ CẢNH này, không phải định nghĩa từ điển.
    Nếu là internet slang/meme → note phải giải thích cultural layer.
    Source: $sourceType
""".trimIndent()
// Model: gemma-4-31b-it | temperature: 0.2 | maxOutputTokens: 350
```

### Khi nào dùng gì

| Task | Approach | Lý do |
|---|---|---|
| Instant explanation (từ phổ thông) | SQLite dictionary | <50ms, offline, curated |
| Context-aware explanation (slang/mới) | Gemini cloud | Hiểu cultural layer |
| Explanation đã tra trước | Room cache | Instant, không tốn quota |
| Pre-warm cho content mới | Gemini server-side | Chạy trước user thấy content |
| Tách từ | jieba (NLP sidecar) | Deterministic, proven |
| Pinyin | pypinyin (NLP sidecar) | Rule-based, 100% chính xác |
| Detect từ khó trong content | jieba + wordlist filter | Không cần AI |
| SRS scheduling | FSRS algorithm | Math thuần túy |
| Content recommendation | Weighted rule scoring | Đủ cho MVP |

### Cost estimate thực tế

```
Build dictionary (một lần):
  15,000 từ × ~200 tokens = 3M tokens input
  Gemini 2.0 Flash: $0.075/1M tokens → ~$0.23 tổng

Cloud explanation (user key):
  Chi phí user chịu từ free quota
  Gemini 2.0 Flash free tier: 1M tokens/ngày → đủ dùng cá nhân

Server-side warmup (server key):
  20 từ/content × 200 tokens × 10 content/ngày = 40K tokens/ngày
  → ~$0.003/ngày — gần như miễn phí

Server cache hit rate sau tuần đầu: ~75-80%
  → 75-80% request của user nhận response <100ms, không tốn quota
```

### Python NLP Sidecar

```python
@app.post("/hard-words")
def get_hard_words(req: HardWordsRequest, n: int = 20):
    """Từ không có trong COMMON_WORDS (HSK 1-4 + 5000 từ phổ thông web)"""
    seen, hard = set(), []
    for word, pos in pseg.cut(req.text):
        if (len(word) >= 2 and word not in COMMON_WORDS
                and word not in seen
                and any('\u4e00' <= c <= '\u9fff' for c in word)):
            seen.add(word)
            hard.append({"word": word, "priority": _priority(pos)})
    hard.sort(key=lambda x: x["priority"], reverse=True)
    return {"words": [h["word"] for h in hard[:n]]}
```

---

## 7. Hướng dẫn triển khai — Step by step

### Phase 1: MVP (Tuần 1-4)

**Tuần 1 — Foundation:**
- [ ] Setup Android project (Kotlin, Compose, Hilt, Room, Retrofit)
- [ ] Setup Spring Boot, Docker Compose (PostgreSQL + Redis + NLP sidecar)
- [ ] Basic auth (JWT) — register/login
- [ ] Seed 20 content items thủ công vào DB
- [ ] ContentFeed screen đọc từ backend

**Tuần 2 — Dictionary + Explanation UI:**
- [ ] Chạy `build_dictionary.py` (để chạy overnight), copy dict vào `assets/`
- [ ] `LocalDictionaryEngine` + unit test lookup performance
- [ ] ReadingScreen với `SelectableChineseText`
- [ ] `ExplainWordUseCase` (dict → cloud flow)
- [ ] `ExplanationBubble` UI + badge Quick/Enhanced
- [ ] Settings screen: API key input

**Tuần 3 — Backend Integration + Save:**
- [ ] `CloudEngine.kt` — Gemini API call
- [ ] Backend `ExplainCacheAPI` — client check server cache trước
- [ ] `CacheWarmupJob` backend chạy với NLP sidecar
- [ ] Save capsule (Room local + sync backend)
- [ ] Offline sync queue

**Tuần 4 — Review + Ship:**
- [ ] FSRS algorithm + LearningState
- [ ] ReviewScreen — Context Recall (4 options)
- [ ] WorkManager push notification
- [ ] Offline graceful handling
- [ ] Bug fix + demo build

**Definition of MVP done:**
- [ ] Highlight từ → explanation <100ms (dict) hoặc loading rõ ràng (cloud)
- [ ] Dict hit rate >70% với content test
- [ ] Full loop: read → highlight → save → review → notification
- [ ] API key lưu secure (EncryptedSharedPreferences)
- [ ] Backend deployed Railway

---

### Phase 2: V1 (Tuần 5-8)

- [ ] ContentIngestion semi-automated (RSS + keyword filter)
- [ ] Topic selection onboarding
- [ ] Learning progress view ("bản đồ chủ đề")
- [ ] Dictionary v2: mở rộng lên 25k từ, bổ sung slang mới
- [ ] A/B test: dict explanation vs cloud explanation — user prefer cái nào?
- [ ] Multi-source content (video transcript)

---

### Phase 3: Scale (sau V1)

- [ ] pgvector: context similarity search
- [ ] On-device LLM revisit — nếu có model <2s trên Android mid-range
- [ ] Fine-tune small model với (word + context → explanation) pairs từ Gemini
- [ ] Collaborative filtering: "user tương tự lưu từ nào"
- [ ] iOS port (KMM cho domain/data layer)

---

## 8. Tech Stack

### Android
```
Language:       Kotlin 2.0
UI:             Jetpack Compose + Material 3
Architecture:   Clean Architecture + MVI
DI:             Hilt
Async:          Coroutines + Flow
Local DB:       Room 2.6 (capsules, explain cache, content)
Local Dict:     SQLite native — bundled in APK assets (~30MB)
Network:        Retrofit 2 + OkHttp 4
Prefs:          DataStore + EncryptedSharedPreferences (API key)
Background:     WorkManager
Testing:        JUnit 5 + MockK + Turbine
```

*Không có MediaPipe / on-device LLM trong MVP. Có thể thêm lại ở Phase 3 khi có model thực sự <2s trên mid-range Android.*

### Backend
```
Language:       Java 21 (Virtual Threads — Project Loom)
Framework:      Spring Boot 3.3
ORM:            Spring Data JPA + Hibernate
Migration:      Flyway
Auth:           Spring Security + JWT (jjwt)
Cache:          Spring Cache + Redis (Lettuce)
Job:            Spring @Scheduled
HTTP Client:    WebClient (Gemini warmup calls)
Build:          Gradle Kotlin DSL
Testing:        JUnit 5 + Testcontainers
```

### Database
```
Primary:        PostgreSQL 16
Cache:          Redis 7 (explain cache, rate limiting)
Local dict:     SQLite (trong APK, read-only, ~30MB)
Embeddings:     pgvector (Phase 2)
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
Deploy:         Railway.app
Container:      Docker + Docker Compose
CDN:            Cloudflare R2 (lưu dict updates cho APK delta)
Monitoring:     Sentry + Grafana Cloud (free tier)
CI/CD:          GitHub Actions
```

---

## 9. Trade-offs & Risks

### Rủi ro 1: Dictionary coverage thấp với slang mới
**Xác suất:** Trung bình  
**Impact:** Trung bình — từ miss → user thấy loading đợi cloud (~3-8s)  
**Giảm thiểu:**
- Cloud xử lý được mọi từ; dict chỉ là fast path, không phải blocker
- Update dict định kỳ (theo quý), thêm slang mới vào `words.txt`
- Analytics: track "dict miss rate" — nếu >30% → update dict sớm hơn

### Rủi ro 2: Gemini API key friction với user
**Xác suất:** Cao — điểm ma sát lớn nhất trong onboarding  
**Impact:** Cao — không có key → chỉ dict + server cache  
**Giảm thiểu:**
- Server warmup cover ~75-80% requests → user dùng được ngay không cần key
- Onboarding: deeplink trực tiếp `aistudio.google.com`, walkthrough 3 bước
- "Enhanced" badge rõ ràng: user hiểu không có key vẫn dùng được

### Rủi ro 3: CacheWarmup miss content
**Xác suất:** Thấp nếu scheduling đúng  
**Impact:** Thấp — fallback về user key hoặc dict  
**Giảm thiểu:**
- Warmup chạy 2h trước push
- Retry với exponential backoff
- Monitor warmup coverage per content item

### Rủi ro 4: Content curation không scale
**Xác suất:** Cao (long-term)  
**Impact:** Trung bình  
**Giảm thiểu:**
- Tuần 1-8: founder tự curate 200-300 items
- Phase 2: semi-automated RSS pipeline
- Phase 3: community submission + moderation

### Rủi ro 5: LLM output không consistent
**Xác suất:** Thấp với temperature 0.2  
**Impact:** Thấp  
**Giảm thiểu:** Parse JSON strict, fallback heuristic, log malformed rate

### Bottleneck khi scale

| Bottleneck | Xảy ra khi | Giải pháp |
|---|---|---|
| PostgreSQL slow | >10k MAU | Read replicas + PgBouncer |
| Redis cache miss | Content mới chưa warm | Tăng warmup window lên 6h |
| Gemini rate limit | >100 req/min warmup | Queue + exponential backoff |
| APK size tăng vì dict | Dict >50MB | Split theo topic, lazy load |

---

## 10. Developer Execution Plan (1 dev, 4 tuần)

### Nguyên tắc

**Không over-engineer** = Đừng build gì tuần 1 chưa cần.  
**Vertical slices** = Mỗi ngày ship feature nhỏ chạy được end-to-end.  
**Dict overnight** = Kick off `build_dictionary.py` đầu tuần 2, để chạy overnight.

---

### Tuần 1 — Foundation

| Ngày | Task | Output có thể test |
|---|---|---|
| T2 | Setup Android (Hilt, Room, Compose nav) + Spring Boot + docker-compose up | Cả hai chạy local |
| T3 | Auth backend (JWT) + Android login/register | Đăng ký, đăng nhập được |
| T4 | Seed 10 content items, ContentAPI, FeedScreen | Đọc được list content |
| T5 | ReadingScreen + SelectableChineseText | Bôi đen text được |
| T6 | Buffer: refactor, test pass, chuẩn bị tuần 2 | Code sạch |

---

### Tuần 2 — Dictionary + Explanation UI

| Ngày | Task | Output có thể test |
|---|---|---|
| T2 | Kick off `build_dictionary.py` overnight (đầu buổi tối T2) | Script chạy |
| T3 | Copy dict vào assets, `LocalDictionaryEngine`, unit test | Lookup <5ms |
| T4 | `ExplainWordUseCase` (dict-only trước) + `ExplanationBubble` | Highlight → dict result |
| T5 | Settings screen: API key input + `CloudEngine` basic | Nhập key, cloud call hoạt động |
| T6 | Hybrid flow: dict → cloud enhance, badge UI | Quick rồi Enhanced |

---

### Tuần 3 — Backend Integration + Capsule

| Ngày | Task | Output có thể test |
|---|---|---|
| T2 | Backend `ExplainCacheAPI` + Android check server cache | Cache hit không tốn quota |
| T3 | `CacheWarmupJob` + NLP sidecar `/hard-words` | Words pre-computed tự động |
| T4 | Save capsule (Room local) | User lưu được từ |
| T5 | Sync capsules backend + offline queue | Sync hoạt động khi có mạng |
| T6 | Dict miss rate logging, analytics cơ bản | Có số liệu để validate |

---

### Tuần 4 — Review System + Ship

| Ngày | Task | Output có thể test |
|---|---|---|
| T2 | `FSRSAlgorithm` unit test + LearningState | FSRS math đúng |
| T3 | ReviewScreen — Context Recall (4 options + reveal) | Full review flow |
| T4 | WorkManager notification khi có từ cần ôn | Nhận notification |
| T5 | Offline mode graceful, error states rõ ràng | App không crash offline |
| T6 | Polish, bug fix, build demo APK | Ship cho 5 người test |

---

### Checklist trước khi ship

- [ ] Dict hit rate >70% với 20 content items test (log và verify)
- [ ] Dict lookup <5ms cho 99% trường hợp (unit test)
- [ ] Cloud explanation trong <10s với Gemini key hợp lệ
- [ ] Full loop hoạt động: read → highlight → save → review → notification
- [ ] API key lưu secure, không bao giờ xuất hiện trong log
- [ ] App không crash khi offline (dict available, cloud hiện lỗi rõ ràng)
- [ ] Backend deployed Railway, health check pass
- [ ] CacheWarmup job chạy thành công ít nhất một lần

---

## Appendix: Quick Start

```bash
# 1. Start tất cả services
docker compose up -d
curl http://localhost:8080/actuator/health  # → {"status":"UP"}

# 2. Build dictionary (một lần, chạy overnight)
pip install google-generativeai jieba pypinyin
python nlp-sidecar/build_dictionary.py \
  --api-key AIza... \
  --word-list nlp-sidecar/words.txt \
  --output chenjin_dict.db
cp chenjin_dict.db android/app/src/main/assets/
echo "1" > android/app/src/main/assets/dict_version.txt

# 3. Android setup
# app/src/debug/res/values/config.xml:
# <string name="backend_url">http://10.0.2.2:8080/</string>
cd android && ./gradlew assembleDebug
```

**Project structure:**
```
chenjin/
├── android/app/src/main/
│   ├── assets/
│   │   ├── chenjin_dict.db          # SQLite dictionary (~30MB)
│   │   └── dict_version.txt         # "1"
│   └── kotlin/com/chenjin/
│       ├── ai/ondevice/LocalDictionaryEngine.kt
│       ├── ai/cloud/CloudEngine.kt
│       ├── domain/usecase/ExplainWordUseCase.kt
│       └── domain/srs/FSRSAlgorithm.kt
├── backend/                          # Java 21 + Spring Boot 3.3
├── nlp-sidecar/
│   ├── main.py                       # tokenize, hard-words endpoints
│   ├── build_dictionary.py           # one-time dict generation
│   └── words.txt                     # seed vocabulary (15k từ)
└── docker-compose.yml
```

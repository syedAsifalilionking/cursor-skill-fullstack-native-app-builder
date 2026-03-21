# OpenAI / ChatGPT Android Integration Reference

Patterns for integrating OpenAI chat and vision APIs via OkHttp + Gson.

## API Key

Fetch from Firebase Remote Config — never hardcode:

```kotlin
val apiKey = AppRemoteConfig.openAIAPIKey
```

If key is empty, suspend-refresh once:

```kotlin
val apiKey = AppRemoteConfig.suspendRefreshOnce()
```

## Data Models

```kotlin
data class ChatGPTMessage(val role: String, val content: String)

data class OpenAIChatRequest(
    val model: String,
    val messages: List<ChatGPTMessage>,
    val temperature: Double = 0.7,
    val max_tokens: Int = 2000
)

data class OpenAIChatResponse(
    val choices: List<Choice>
) {
    data class Choice(val message: Message) {
        data class Message(val content: String)
    }
}

data class OpenAIAPIError(
    val error: ErrorBody
) {
    data class ErrorBody(val message: String, val type: String?)
}
```

**ProGuard:** Keep these classes from obfuscation:

```proguard
-keep class com.yourpackage.**.ChatGPTMessage { *; }
-keep class com.yourpackage.**.OpenAIChatRequest { *; }
-keep class com.yourpackage.**.OpenAIChatResponse { *; }
-keep class com.yourpackage.**.OpenAIAPIError { *; }
```

## Chat Service

```kotlin
class ChatGPTService {
    private val client = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(60, TimeUnit.SECONDS)
        .build()
    private val gson = Gson()

    private val conversationHistory = mutableListOf<ChatGPTMessage>()

    private val systemPrompt = """
        You are a helpful assistant for this app. Help users with features,
        answer questions, and provide recommendations. Be concise and friendly.
        Respond in the user's language.
    """.trimIndent()

    suspend fun send(userMessage: String): String = withContext(Dispatchers.IO) {
        conversationHistory.add(ChatGPTMessage("user", userMessage))

        val messages = mutableListOf(ChatGPTMessage("system", systemPrompt))
        messages.addAll(conversationHistory.takeLast(20))

        val response = sendRequest("gpt-4o", messages)
        val reply = response.choices.firstOrNull()?.message?.content ?: ""
        conversationHistory.add(ChatGPTMessage("assistant", reply))
        reply
    }

    fun clearHistory() { conversationHistory.clear() }

    private suspend fun sendRequest(model: String, messages: List<ChatGPTMessage>): OpenAIChatResponse {
        val apiKey = AppRemoteConfig.openAIAPIKey
        require(apiKey.isNotEmpty()) { "OpenAI API key not configured" }

        val requestBody = OpenAIChatRequest(model = model, messages = messages)
        val json = gson.toJson(requestBody)

        val request = Request.Builder()
            .url("https://api.openai.com/v1/chat/completions")
            .addHeader("Authorization", "Bearer $apiKey")
            .addHeader("Content-Type", "application/json")
            .post(json.toRequestBody("application/json".toMediaType()))
            .build()

        return performWithRetry(request)
    }

    private suspend fun performWithRetry(request: Request, maxRetries: Int = 3): OpenAIChatResponse {
        var lastError: Exception? = null
        repeat(maxRetries) { attempt ->
            try {
                val response = client.newCall(request).execute()
                val body = response.body?.string() ?: throw Exception("Empty response")

                if (response.code == 429 || response.code >= 500) {
                    delay((attempt + 1) * 2000L)
                    return@repeat
                }

                if (!response.isSuccessful) {
                    val error = runCatching { gson.fromJson(body, OpenAIAPIError::class.java) }.getOrNull()
                    throw Exception(error?.error?.message ?: "API error ${response.code}")
                }

                return gson.fromJson(body, OpenAIChatResponse::class.java)
            } catch (e: Exception) {
                lastError = e
                if (attempt < maxRetries - 1) delay((attempt + 1) * 2000L)
            }
        }
        throw lastError ?: Exception("Unknown error")
    }
}
```

## Vision Service (Image Analysis)

```kotlin
class VisionGPTService {
    private val client = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(90, TimeUnit.SECONDS)
        .build()
    private val gson = Gson()

    suspend fun analyzeImage(bitmap: Bitmap, prompt: String): String = withContext(Dispatchers.IO) {
        val apiKey = AppRemoteConfig.openAIAPIKey
        require(apiKey.isNotEmpty()) { "API key not configured" }

        val stream = ByteArrayOutputStream()
        bitmap.compress(Bitmap.CompressFormat.JPEG, 70, stream)
        val base64 = Base64.encodeToString(stream.toByteArray(), Base64.NO_WRAP)
        val imageUrl = "data:image/jpeg;base64,$base64"

        val messagesJson = JsonArray().apply {
            add(JsonObject().apply {
                addProperty("role", "system")
                addProperty("content", "You are an image analysis assistant. Respond in JSON.")
            })
            add(JsonObject().apply {
                addProperty("role", "user")
                add("content", JsonArray().apply {
                    add(JsonObject().apply {
                        addProperty("type", "text")
                        addProperty("text", prompt)
                    })
                    add(JsonObject().apply {
                        addProperty("type", "image_url")
                        add("image_url", JsonObject().apply {
                            addProperty("url", imageUrl)
                        })
                    })
                })
            })
        }

        val requestJson = JsonObject().apply {
            addProperty("model", "gpt-4o")
            add("messages", messagesJson)
            addProperty("max_tokens", 2000)
        }

        val request = Request.Builder()
            .url("https://api.openai.com/v1/chat/completions")
            .addHeader("Authorization", "Bearer $apiKey")
            .addHeader("Content-Type", "application/json")
            .post(requestJson.toString().toRequestBody("application/json".toMediaType()))
            .build()

        val response = client.newCall(request).execute()
        val body = response.body?.string() ?: throw Exception("Empty response")

        if (!response.isSuccessful) {
            val error = runCatching { gson.fromJson(body, OpenAIAPIError::class.java) }.getOrNull()
            throw Exception(error?.error?.message ?: "API error ${response.code}")
        }

        val parsed = gson.fromJson(body, OpenAIChatResponse::class.java)
        parsed.choices.firstOrNull()?.message?.content ?: throw Exception("No response content")
    }
}
```

## Structured JSON Output

```kotlin
val systemPrompt = """
Analyze the image and respond ONLY with valid JSON:
{
    "items": [
        { "name": "...", "category": "...", "confidence": 0.95 }
    ]
}
""".trimIndent()

// Parse response
data class AnalysisResult(val items: List<AnalysisItem>)
data class AnalysisItem(val name: String, val category: String, val confidence: Double)

val result = gson.fromJson(responseContent, AnalysisResult::class.java)
```

## AI Recommendations Service

```kotlin
class AIRecommendationsService {
    private val prefs by lazy { context.getSharedPreferences("ai_recs", Context.MODE_PRIVATE) }

    suspend fun fetchIfNeeded(userProfile: Map<String, Any>): List<Recommendation> {
        val today = SimpleDateFormat("yyyy-MM-dd", Locale.US).format(Date())
        if (prefs.getString("last_fetch", "") == today) {
            return loadCached()
        }

        val prompt = buildPromptFrom(userProfile)
        val response = chatService.sendRequest("gpt-4o", listOf(
            ChatGPTMessage("system", "Generate personalized recommendations. Respond in JSON array."),
            ChatGPTMessage("user", prompt)
        ))

        val recommendations = parseRecommendations(response)
        saveCached(recommendations)
        prefs.edit().putString("last_fetch", today).apply()
        return recommendations
    }
}
```

## Multiple AI Services Pattern

| Service | Model | Purpose |
|---------|-------|---------|
| ChatGPTService | gpt-4o | Interactive assistant |
| VisionGPTService | gpt-4o | Image analysis (camera scans) |
| RecommendationsService | gpt-4o | Daily personalized suggestions |
| PlanGeneratorService | gpt-4o | Generate plans/schedules |

Each service:
- Checks `AIConsentManager.hasConsented(uid)` before calls
- Gets API key from `AppRemoteConfig.openAIAPIKey`
- Has its own system prompt
- Implements retry with exponential backoff (429, 5xx)

## Conversation History Persistence

```kotlin
fun saveHistory(context: Context) {
    val json = gson.toJson(conversationHistory)
    context.getSharedPreferences("chat", Context.MODE_PRIVATE)
        .edit().putString("history", json).apply()
}

fun loadHistory(context: Context) {
    val json = context.getSharedPreferences("chat", Context.MODE_PRIVATE)
        .getString("history", null) ?: return
    val type = object : TypeToken<List<ChatGPTMessage>>() {}.type
    conversationHistory.clear()
    conversationHistory.addAll(gson.fromJson(json, type))
}
```

## Error Handling

```kotlin
sealed class AIError(message: String) : Exception(message) {
    class ConsentRequired : AIError("AI consent required")
    class MissingAPIKey : AIError("API key not configured")
    class RateLimited : AIError("Rate limited, try again later")
    class APIError(msg: String) : AIError(msg)
}
```

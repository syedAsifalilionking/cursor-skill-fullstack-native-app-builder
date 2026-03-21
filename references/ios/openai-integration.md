# OpenAI / ChatGPT Integration Reference

Patterns for integrating OpenAI's chat and vision APIs into iOS apps via URLSession.

## API Key Management

Never hardcode API keys. Fetch from Firebase Remote Config:

```swift
let apiKey = AppRemoteConfig.shared.openAIAPIKey
```

## Data Models

```swift
struct ChatGPTMessage: Codable {
    let role: String   // "system", "user", "assistant"
    let content: String
}

struct ChatGPTRequest: Codable {
    let model: String
    let messages: [ChatGPTMessage]
    let temperature: Double?
    let max_tokens: Int?
}

struct ChatGPTResponse: Codable {
    struct Choice: Codable {
        struct Message: Codable { let content: String }
        let message: Message
    }
    let choices: [Choice]
}

struct OpenAIAPIError: Codable {
    struct ErrorBody: Codable { let message: String; let type: String? }
    let error: ErrorBody
}
```

## Chat Service (Text)

```swift
final class ChatGPTService: ObservableObject {
    static let shared = ChatGPTService()
    @Published var messages: [ChatGPTMessage] = []
    @Published var isLoading = false

    private let systemPrompt = """
    You are a helpful assistant for this app. You help users with features,
    answer questions, and provide recommendations. Be concise and friendly.
    Respond in the user's language.
    """

    func send(_ userMessage: String) async throws -> String {
        guard AIConsentManager.shared.hasConsented else {
            throw AIError.consentRequired
        }

        messages.append(ChatGPTMessage(role: "user", content: userMessage))

        var requestMessages = [ChatGPTMessage(role: "system", content: systemPrompt)]
        requestMessages.append(contentsOf: messages.suffix(20)) // conversation window

        let response = try await sendRequest(
            model: "gpt-4o",
            messages: requestMessages
        )

        let reply = response.choices.first?.message.content ?? ""
        messages.append(ChatGPTMessage(role: "assistant", content: reply))
        return reply
    }

    func clearHistory() { messages.removeAll() }
}
```

## Vision Service (Image Analysis)

```swift
final class VisionGPTService {
    static let shared = VisionGPTService()

    func analyzeImage(_ image: UIImage, prompt: String) async throws -> String {
        guard AIConsentManager.shared.hasConsented else {
            throw AIError.consentRequired
        }

        guard let jpegData = image.jpegData(compressionQuality: 0.7) else {
            throw AIError.invalidImage
        }
        let base64 = jpegData.base64EncodedString()
        let imageUrl = "data:image/jpeg;base64,\(base64)"

        let userContent: [[String: Any]] = [
            ["type": "text", "text": prompt],
            ["type": "image_url", "image_url": ["url": imageUrl]]
        ]

        let requestBody: [String: Any] = [
            "model": "gpt-4o",
            "messages": [
                ["role": "system", "content": "You are an image analysis assistant. Respond in JSON."],
                ["role": "user", "content": userContent]
            ],
            "max_tokens": 2000
        ]

        return try await sendRawRequest(body: requestBody)
    }
}
```

## Shared Network Layer

```swift
private func sendRequest(model: String, messages: [ChatGPTMessage], temperature: Double = 0.7) async throws -> ChatGPTResponse {
    let apiKey = AppRemoteConfig.shared.openAIAPIKey
    guard !apiKey.isEmpty else { throw AIError.missingAPIKey }

    let request = ChatGPTRequest(model: model, messages: messages, temperature: temperature, max_tokens: 2000)
    let body = try JSONEncoder().encode(request)

    var urlRequest = URLRequest(url: URL(string: "https://api.openai.com/v1/chat/completions")!)
    urlRequest.httpMethod = "POST"
    urlRequest.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
    urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
    urlRequest.httpBody = body

    return try await performWithRetry(urlRequest)
}

private func performWithRetry<T: Decodable>(_ request: URLRequest, maxRetries: Int = 3) async throws -> T {
    var lastError: Error?
    for attempt in 0..<maxRetries {
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            let httpResponse = response as? HTTPURLResponse

            if let statusCode = httpResponse?.statusCode, statusCode == 429 || statusCode >= 500 {
                let delay = Double(attempt + 1) * 2.0
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                continue
            }

            if let apiError = try? JSONDecoder().decode(OpenAIAPIError.self, from: data) {
                throw AIError.apiError(apiError.error.message)
            }

            return try JSONDecoder().decode(T.self, from: data)
        } catch {
            lastError = error
            if attempt < maxRetries - 1 {
                try await Task.sleep(nanoseconds: UInt64((attempt + 1) * 2) * 1_000_000_000)
            }
        }
    }
    throw lastError ?? AIError.unknown
}
```

## Structured JSON Output

For AI features that need structured data, instruct the model to respond in JSON:

```swift
let systemPrompt = """
Analyze the image and respond ONLY with valid JSON in this format:
{
    "items": [
        {
            "name": "Item name",
            "category": "Category",
            "confidence": 0.95,
            "details": { ... }
        }
    ]
}
"""
```

Parse the response:

```swift
struct AnalysisResult: Codable {
    struct Item: Codable {
        let name: String
        let category: String
        let confidence: Double
        let details: [String: String]?
    }
    let items: [Item]
}

let result = try JSONDecoder().decode(AnalysisResult.self, from: responseString.data(using: .utf8)!)
```

## AI Recommendations Service

Generate daily personalized recommendations:

```swift
final class AIRecommendationsService {
    static let shared = AIRecommendationsService()
    private let cacheKey = "ai_recommendations"
    private let lastFetchKey = "ai_recommendations_date"

    func fetchIfNeeded(userProfile: UserProfile) async throws -> [Recommendation] {
        // Check if already fetched today
        if let lastFetch = UserDefaults.standard.string(forKey: lastFetchKey),
           lastFetch == todayString() {
            return LocalCacheManager.shared.load([Recommendation].self, forKey: cacheKey) ?? []
        }

        let prompt = buildPrompt(from: userProfile)
        let response = try await sendRequest(model: "gpt-4o", messages: [
            ChatGPTMessage(role: "system", content: "Generate personalized recommendations. Respond in JSON array."),
            ChatGPTMessage(role: "user", content: prompt)
        ])

        let recommendations = try parse(response)
        LocalCacheManager.shared.save(recommendations, forKey: cacheKey)
        UserDefaults.standard.set(todayString(), forKey: lastFetchKey)
        return recommendations
    }
}
```

## Multiple AI Service Pattern

Create separate service classes for different AI features:

| Service | Model | Purpose |
|---------|-------|---------|
| ChatGPTService | gpt-4o | Interactive chat assistant |
| VisionGPTService | gpt-4o | Image analysis (camera scans) |
| RecommendationsService | gpt-4o | Daily personalized recommendations |
| PlanGeneratorService | gpt-4o | Generate plans/schedules |

Each service:
- Checks `AIConsentManager.shared.hasConsented` before any call
- Uses `AppRemoteConfig.shared.openAIAPIKey`
- Has its own system prompt tailored to the feature
- Implements retry with exponential backoff (429, 5xx)

## AI Feature Gating

Gate AI features behind subscription + consent:

```swift
struct AIFeatureDisabledView: View {
    let reason: AIGateReason

    enum AIGateReason {
        case consentRequired
        case subscriptionRequired
        case limitReached
    }

    var body: some View {
        VStack {
            Image(systemName: "sparkles")
            Text(message)
            Button(actionTitle) { handleAction() }
        }
    }
}
```

## Conversation History

Persist chat history in UserDefaults for session continuity:

```swift
private func saveHistory() {
    if let data = try? JSONEncoder().encode(messages) {
        UserDefaults.standard.set(data, forKey: "conversationHistory")
    }
}

private func loadHistory() {
    if let data = UserDefaults.standard.data(forKey: "conversationHistory"),
       let saved = try? JSONDecoder().decode([ChatGPTMessage].self, from: data) {
        messages = saved
    }
}
```

## Error Handling

```swift
enum AIError: LocalizedError {
    case consentRequired
    case missingAPIKey
    case invalidImage
    case apiError(String)
    case rateLimited
    case unknown

    var errorDescription: String? {
        switch self {
        case .consentRequired: return String(localized: "ai_consent_required")
        case .missingAPIKey: return String(localized: "ai_config_error")
        case .invalidImage: return String(localized: "ai_invalid_image")
        case .apiError(let msg): return msg
        case .rateLimited: return String(localized: "ai_rate_limited")
        case .unknown: return String(localized: "ai_unknown_error")
        }
    }
}
```

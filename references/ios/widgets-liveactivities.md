# Widgets & Live Activities Reference

Patterns for Home Screen widgets, Lock Screen widgets, and Live Activities with App Groups for data sharing.

## App Group Setup

1. Enable App Groups in both main app and widget extension targets
2. Use shared `UserDefaults(suiteName:)` for data passing

```swift
// Shared constants
enum AppGroupConstants {
    static let suiteName = "group.com.yourapp.shared"
}
```

## Widget Data Bridge (Main App Side)

Write data from the main app for widgets to read:

```swift
final class WidgetDataBridge {
    static let shared = WidgetDataBridge()
    private let defaults = UserDefaults(suiteName: AppGroupConstants.suiteName)!

    // MARK: - Write Methods

    func updateProgress(current: Int, goal: Int) {
        defaults.set(current, forKey: "progressCurrent")
        defaults.set(goal, forKey: "progressGoal")
        defaults.set(Date().timeIntervalSince1970, forKey: "lastUpdated")
        WidgetCenter.shared.reloadAllTimelines()
    }

    func updateStreak(current: Int, longest: Int) {
        defaults.set(current, forKey: "currentStreak")
        defaults.set(longest, forKey: "longestStreak")
        defaults.set(Date().timeIntervalSince1970, forKey: "lastUpdated")
        WidgetCenter.shared.reloadAllTimelines()
    }

    func updateCounter(value: Int, target: Int) {
        defaults.set(value, forKey: "counterValue")
        defaults.set(target, forKey: "counterTarget")
        defaults.set(Date().timeIntervalSince1970, forKey: "lastUpdated")
        WidgetCenter.shared.reloadAllTimelines()

        // Also update Live Activity if running
        SomeLiveActivityManager.shared.startOrUpdate(current: value, goal: target)
    }
}
```

## Widget Data Reader (Widget Extension Side)

```swift
struct WidgetDataReader {
    private static let defaults = UserDefaults(suiteName: AppGroupConstants.suiteName)!

    static var progressCurrent: Int { defaults.integer(forKey: "progressCurrent") }
    static var progressGoal: Int { defaults.integer(forKey: "progressGoal") }
    static var currentStreak: Int { defaults.integer(forKey: "currentStreak") }
    static var longestStreak: Int { defaults.integer(forKey: "longestStreak") }
    static var counterValue: Int { defaults.integer(forKey: "counterValue") }
    static var counterTarget: Int { defaults.integer(forKey: "counterTarget") }

    static var lastUpdated: Date {
        Date(timeIntervalSince1970: defaults.double(forKey: "lastUpdated"))
    }
}
```

## Widget Bundle

```swift
@main
struct AppWidgetBundle: WidgetBundle {
    var body: some Widget {
        ProgressWidget()
        CounterWidget()
        StreakWidget()

        // Live Activities
        if #available(iOS 16.2, *) {
            ProgressLiveActivityWidget()
            CounterLiveActivityWidget()
        }
    }
}
```

## Home Screen Widget

```swift
struct ProgressWidget: Widget {
    let kind = "ProgressWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: ProgressProvider()) { entry in
            ProgressWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Progress")
        .description("Track your daily progress.")
        .supportedFamilies([.systemSmall, .systemMedium, .accessoryCircular, .accessoryRectangular])
    }
}

struct ProgressEntry: TimelineEntry {
    let date: Date
    let current: Int
    let goal: Int
    var progress: Double { goal > 0 ? min(Double(current) / Double(goal), 1.0) : 0 }
}

struct ProgressProvider: TimelineProvider {
    func placeholder(in context: Context) -> ProgressEntry {
        ProgressEntry(date: .now, current: 50, goal: 100)
    }

    func getSnapshot(in context: Context, completion: @escaping (ProgressEntry) -> Void) {
        completion(ProgressEntry(
            date: .now,
            current: WidgetDataReader.progressCurrent,
            goal: WidgetDataReader.progressGoal
        ))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<ProgressEntry>) -> Void) {
        let entry = ProgressEntry(
            date: .now,
            current: WidgetDataReader.progressCurrent,
            goal: WidgetDataReader.progressGoal
        )
        let nextUpdate = Calendar.current.date(byAdding: .minute, value: 15, to: .now)!
        completion(Timeline(entries: [entry], policy: .after(nextUpdate)))
    }
}

struct ProgressWidgetView: View {
    var entry: ProgressEntry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            VStack {
                ZStack {
                    Circle().stroke(Color.gray.opacity(0.2), lineWidth: 8)
                    Circle().trim(from: 0, to: entry.progress)
                        .stroke(Color.blue, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                        .rotationEffect(.degrees(-90))
                    Text("\(Int(entry.progress * 100))%").font(.title2.bold())
                }
                Text("\(entry.current)/\(entry.goal)").font(.caption)
            }
            .padding()

        case .accessoryCircular:
            Gauge(value: entry.progress) {
                Text("\(entry.current)")
            }
            .gaugeStyle(.accessoryCircularCapacity)

        default:
            HStack {
                ProgressView(value: entry.progress)
                    .progressViewStyle(.linear)
                Text("\(entry.current)/\(entry.goal)")
            }
            .padding()
        }
    }
}
```

## Streak Widget

```swift
struct StreakWidget: Widget {
    let kind = "StreakWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: StreakProvider()) { entry in
            StreakWidgetView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("Streak")
        .description("Track your daily streak.")
        .supportedFamilies([.systemSmall, .accessoryRectangular])
    }
}

struct StreakEntry: TimelineEntry {
    let date: Date
    let current: Int
    let longest: Int
}

struct StreakProvider: TimelineProvider {
    func placeholder(in context: Context) -> StreakEntry {
        StreakEntry(date: .now, current: 7, longest: 30)
    }

    func getSnapshot(in context: Context, completion: @escaping (StreakEntry) -> Void) {
        completion(StreakEntry(date: .now, current: WidgetDataReader.currentStreak, longest: WidgetDataReader.longestStreak))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<StreakEntry>) -> Void) {
        let entry = StreakEntry(date: .now, current: WidgetDataReader.currentStreak, longest: WidgetDataReader.longestStreak)
        let midnight = Calendar.current.startOfDay(for: Calendar.current.date(byAdding: .day, value: 1, to: .now)!)
        completion(Timeline(entries: [entry], policy: .after(midnight)))
    }
}
```

## Live Activity

### Activity Attributes

Define in BOTH main app and widget extension targets:

```swift
import ActivityKit

struct ProgressActivityAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        var current: Int
        var goal: Int
        var progress: Double { goal > 0 ? min(Double(current) / Double(goal), 1.0) : 0 }
    }

    var label: String
}
```

### Live Activity Manager (Main App)

```swift
@MainActor
final class ProgressLiveActivityManager {
    static let shared = ProgressLiveActivityManager()
    private var activity: Activity<ProgressActivityAttributes>?

    func startOrUpdate(current: Int, goal: Int) {
        guard ActivityAuthorizationInfo().areActivitiesEnabled else { return }

        let state = ProgressActivityAttributes.ContentState(current: current, goal: goal)

        if let existing = activity, existing.activityState == .active {
            Task { await existing.update(ActivityContent(state: state, staleDate: nil)) }
        } else {
            let attributes = ProgressActivityAttributes(label: "Progress")
            let content = ActivityContent(state: state, staleDate: nil)
            activity = try? Activity.request(attributes: attributes, content: content, pushType: nil)
        }
    }

    func end() {
        guard let activity else { return }
        let finalState = ProgressActivityAttributes.ContentState(current: 0, goal: 0)
        Task { await activity.end(ActivityContent(state: finalState, staleDate: nil), dismissalPolicy: .immediate) }
        self.activity = nil
    }
}
```

### Live Activity Widget (Widget Extension)

```swift
@available(iOS 16.2, *)
struct ProgressLiveActivityWidget: Widget {
    var body: some WidgetConfiguration {
        ActivityConfiguration(for: ProgressActivityAttributes.self) { context in
            // Lock Screen banner
            HStack {
                VStack(alignment: .leading) {
                    Text(context.attributes.label).font(.headline)
                    Text("\(context.state.current) / \(context.state.goal)").font(.caption)
                }
                Spacer()
                ZStack {
                    Circle().stroke(Color.gray.opacity(0.3), lineWidth: 4)
                    Circle().trim(from: 0, to: context.state.progress)
                        .stroke(Color.blue, lineWidth: 4)
                        .rotationEffect(.degrees(-90))
                    Text("\(Int(context.state.progress * 100))%").font(.caption2)
                }
                .frame(width: 44, height: 44)
            }
            .padding()

        } dynamicIsland: { context in
            DynamicIsland {
                DynamicIslandExpandedRegion(.leading) {
                    Text(context.attributes.label).font(.headline)
                }
                DynamicIslandExpandedRegion(.trailing) {
                    Text("\(context.state.current)/\(context.state.goal)")
                }
                DynamicIslandExpandedRegion(.bottom) {
                    ProgressView(value: context.state.progress)
                        .progressViewStyle(.linear)
                }
            } compactLeading: {
                Image(systemName: "chart.bar.fill")
            } compactTrailing: {
                Text("\(Int(context.state.progress * 100))%")
            } minimal: {
                Image(systemName: "chart.bar.fill")
            }
        }
    }
}
```

## Info.plist for Live Activities

Add to main app's `Info.plist`:

```xml
<key>NSSupportsLiveActivities</key>
<true/>
```

## Entitlements

Both targets need the same App Group:

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>group.com.yourapp.shared</string>
</array>
```

## Deep Link from Widget

```swift
// In widget view
Link(destination: URL(string: "appname://someaction")!) {
    WidgetContent()
}

// Or via widgetURL for the entire widget
.widgetURL(URL(string: "appname://someaction"))
```

Handle in main app via `.onOpenURL { url in ... }`.

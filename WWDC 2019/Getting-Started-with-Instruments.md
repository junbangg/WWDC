# Time Profiler

Collects call stacks of relevant threads at a fixed interval

# Points of Interest

Collects important data from your app that you can highlight 

# Profiling Tips

- Time Profiler shows how your app is spending time
- Check main thread when responsiveness issues occur
- Profile release builds
    - profile in release mode
    - Build-Run 할때는 optimization 이 있다
    - 이건 shipping 할때는 없음
    - 그래서 release mode 로 확인해볼것
- Profile with difficult workloads and older devices

# Simulator Caveat

- real hardware

# Signposts

- Simpler and more efficient than printing
- Built in support for measuring time
- Traced by instruments

`print` 문을 → `OSLog` 로 대체하여 points of interest 로 확인가능 

```swift
static let pointsOfInterest = OSLog(subsystem: "com.apple.SolarSystem", category: .pointsOfInterest)

os_signpost(.begin, log: SceneController.pointsOfInterest, name: "setupScene")
defer {
os_signpost(.end, log: SceneController.pointsOfInterest, name: "setupScene")
}
```

- Statistical profiles show which code is most commonly executed

- Exact measurements show how and why code is executed

- XCTests reliably reproduce workloads for profiling

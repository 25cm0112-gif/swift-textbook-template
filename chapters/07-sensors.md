# 第7章：センサーの活用

> 執筆者：呉迎豪
> 最終更新：2026-7-1

## この章で学ぶこと
この章では、SwiftUIで歩数や位置情報を取得し、活動データをリアルタイムに表示する方法を学ぶ。また、タイマーや速度表示を組み合わせて、活動トラッカーアプリを作成する方法を学ぶ。
## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
import SwiftUI
import CoreMotion
import CoreLocation

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?
    var endTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        endTime = nil
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        endTime = Date()
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()

    // 経過時間（startTime からの差分。計測中は現在時刻 date、停止後は endTime で固定）
    private func elapsedTime(at date: Date) -> TimeInterval {
        guard let start = tracker.startTime else { return 0 }
        let end = tracker.endTime ?? date
        return max(0, end.timeIntervalSince(start))
    }

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        // TimelineView が1秒ごとに context.date を更新し、この中だけが再描画される
        TimelineView(.periodic(from: .now, by: 1)) { context in
            VStack(spacing: 4) {
                Text(formatTime(elapsedTime(at: context.date)))
                    .font(.system(size: 48, weight: .thin, design: .monospaced))

                if tracker.isTracking {
                    Text("計測中")
                        .font(.caption)
                        .foregroundStyle(.green)
                }
            }
            .padding()
        }
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ContentView()
}// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**
このアプリは、iPhoneのセンサーを利用して歩数・移動距離・速度・消費カロリーを計測する活動トラッカーである。スタートボタンを押すと計測を開始し、歩数計とGPSから取得した情報をリアルタイムで画面に表示する。計測中は経過時間や現在の速度を確認でき、ストップボタンを押すと計測を終了する。
## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
private let pedometer = CMPedometer()

pedometer.startUpdates(from: Date()) { [weak self] data, error in
    guard let self = self, let data = data else { return }

    DispatchQueue.main.async {
        self.stepCount = data.numberOfSteps.intValue
        if let dist = data.distance {
            self.distance = dist.doubleValue
        }
    }
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
CMPedometerを利用して歩数と移動距離を取得し、画面へリアルタイムに反映している。
**なぜこう書くのか：**
歩数データはバックグラウンドで取得されるため、画面の更新はDispatchQueue.main.asyncでメインスレッドに戻して行う必要がある。また、[weak self]を使うことでメモリリークを防いでいる。
**もしこう書かなかったら：**
DispatchQueue.main.asyncを書かないと画面が更新されない場合がある。また、[weak self]を省略するとメモリが解放されず、アプリの動作に悪影響を与える可能性がある。
---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
private let locationManager = CLLocationManager()

locationManager.delegate = self
locationManager.requestWhenInUseAuthorization()
locationManager.startUpdatingLocation()// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
GPSを利用して現在位置を取得し、速度や移動経路を計測している。
**なぜこう書くのか：**
位置情報を取得するにはユーザーの許可が必要であるため、requestWhenInUseAuthorization()を呼び出している。また、startUpdatingLocation()で位置情報の更新を開始している。
**もしこう書かなかったら：**
位置情報の許可が得られず、速度や移動経路を取得できない。
---

### 歩数計（CMPedometer）

```swift
func locationManager(_ manager: CLLocationManager,
                     didUpdateLocations newLocations: [CLLocation]) {

    guard let location = newLocations.last else { return }

    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
GPSから取得した最新の位置情報を保存し、現在の速度を更新している。
**なぜこう書くのか：**
newLocations.lastを利用することで最新の位置情報だけを利用できる。また、max(0, location.speed)でマイナスの速度を0に補正している。
**もしこう書かなかったら：**
古い位置情報を使ってしまったり、停止中に速度が-1と表示されることがある。
---

### CoreLocationとの連携

```swift
TimelineView(.periodic(from: .now, by: 1)) { context in
    Text(formatTime(elapsedTime(at: context.date)))
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
1秒ごとに画面を更新し、経過時間をリアルタイムで表示している
**なぜこう書くのか：**
Timerを自分で管理する必要がなく、SwiftUIに適した方法でタイマー表示ができるためである。
**もしこう書かなかったら：**
経過時間が更新されず、スタート時刻のまま表示されてしまう。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目                          | 説明                | 使用例                                       |
| --------------------------- | ----------------- | ----------------------------------------- |
| `CMPedometer`               | 歩数や歩行距離を取得する      | `pedometer.startUpdates(from:)`           |
| `CLLocationManager`         | GPSから位置情報を取得する    | `locationManager.startUpdatingLocation()` |
| `CLLocationManagerDelegate` | 位置情報が更新されたことを受け取る | `didUpdateLocations`                      |
| `TimelineView`              | 一定時間ごとに画面を更新する    | `TimelineView(.periodic(...))`            |
| `@Observable`               | データが変わると画面を自動更新する | `@Observable class ActivityTracker`       |
| `LazyVGrid`                 | グリッド形式で画面を表示する    | `LazyVGrid(columns: ...)`                 |
| `NavigationStack`           | 画面全体を管理する         | `NavigationStack { ... }`                 |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

実験1
やったこと： 速度メーターの最大速度を10km/hに変更した。
結果： 少し歩くだけでメーターが最大まで表示された。
わかったこと： 最大値を適切に設定しないと速度表示が分かりにくくなる。
実験2
やったこと： TimelineViewを削除した。
結果： 経過時間が更新されなくなった。
わかったこと： タイマー表示には定期的な画面更新が必要である。

## AIに聞いて特に理解が深まった質問 TOP3

1. 質問： TimelineViewとTimerの違いは何ですか。

得られた理解： TimelineViewはSwiftUI向けに作られており、効率よく画面を更新できることが分かった。

2. 質問： DispatchQueue.main.asyncはなぜ必要ですか。

得られた理解： バックグラウンドで取得したデータを画面に表示するには、メインスレッドで更新する必要があることが理解できた。

3. 質問： location.speedが-1になる理由は何ですか。

得られた理解： GPSが十分な速度を測定できない場合は-1になるため、max(0, location.speed)で補正していることが分かった。

この章のまとめ

この章では、Core MotionとCore Locationを利用して歩数・移動距離・速度などの活動データを取得する方法を学んだ。また、SwiftUIの@ObservableやTimelineViewを利用して、取得したデータをリアルタイムで画面に表示する方法も学んだ。センサーやGPSを組み合わせることで、実際に利用できる活動トラッカーアプリを作成できることを理解した。

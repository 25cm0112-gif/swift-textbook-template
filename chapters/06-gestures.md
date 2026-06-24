# 第6章：ジェスチャー操作

> 執筆者：呉迎豪
> 最終更新：2026-6-24

## この章で学ぶこと
この章では、SwiftUIでジェスチャーを利用した画面操作について学ぶ。特にドラッグジェスチャーを使ってカードを左右にスワイプする方法や、アニメーションと組み合わせて自然なUIを実装する方法を学ぶ。また、状態管理（@State）を利用して画面を更新する方法も理解する。
## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.first {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    ContentView()
}// ここに模範コード全体を貼る
```

**このアプリは何をするものか：**

このアプリは、表示された動物カードを左右にスワイプして「好き（LIKE）」または「嫌い（NOPE）」を選択するアプリである。右にスワイプすると「好き」、左にスワイプすると「嫌い」としてカウントされる。すべてのカードを選び終えると完了画面が表示され、「もう一度」ボタンで最初からやり直すことができる。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
Button {
    if let top = animals.first {
        removeCard(animal: top, direction: .left)
    }
} label: {
    Image(systemName: "xmark.circle.fill")
}

Button {
    if let top = animals.first {
        removeCard(animal: top, direction: .right)
    }
} label: {
    Image(systemName: "heart.circle.fill")
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
ボタンをタップすると、一番上に表示されているカードを「好き」または「嫌い」として判定し、カードを削除してカウントを更新している。

**なぜこう書くのか：**
Buttonを使うことで、スワイプ操作ができない場合でも同じ処理を実行できる。また、removeCard()を共通で呼び出すことで、処理を一か所にまとめている。

**もしこう書かなかったら：**
ボタンを押しても何も起こらず、スワイプだけでしか操作できなくなる。また、同じ処理を複数回書くことになり、保守がしにくくなる。

---

### ドラッグジェスチャーとオフセット管理

```swift
.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
            rotation = Double(value.translation.width / 20)
        }
        .onEnded { value in
            if value.translation.width > swipeThreshold {
                withAnimation(.easeOut(duration: 0.3)) {
                    offset = CGSize(width: 500, height: 0)
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                    onSwipe(.right)
                }
            } else if value.translation.width < -swipeThreshold {
                withAnimation(.easeOut(duration: 0.3)) {
                    offset = CGSize(width: -500, height: 0)
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                    onSwipe(.left)
                }
            } else {
                withAnimation(.spring) {
                    offset = .zero
                    rotation = 0
                }
            }
        }
)// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
ユーザーがカードをドラッグすると、その移動量に合わせてカードを動かし、少し回転させている。一定以上ドラッグするとカードが画面外へ飛び、左右どちらへスワイプしたかを判定する。
**なぜこう書くのか：**
DragGestureを使うことで、ユーザーの指の動きに合わせてカードをリアルタイムに動かせる。また、offsetを変更することで位置を更新し、rotationでカードを傾けることで自然な動きを表現している。
**もしこう書かなかったら：**
カードはドラッグしても動かず、スワイプ操作ができない。また、回転を付けなければ動きが単調になり、操作感が悪くなる。
---

### 拡大縮小と回転

```swift
.rotationEffect(.degrees(rotation))rotation = Double(value.translation.width / 20)// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
ドラッグした距離に応じてカードを左右に回転させている。
**なぜこう書くのか：**
カードが少し傾くことで、実際に手でカードを動かしているような自然な操作感を演出できる。
**もしこう書かなかったら：**
カードは左右へ移動するだけになり、見た目が不自然で単調なアニメーションになる。
---

### ジェスチャーの組み合わせとアニメーション

```swift
withAnimation(.easeOut(duration: 0.3)) {
    offset = CGSize(width: 500, height: 0)
}withAnimation(.spring) {
    offset = .zero
    rotation = 0
}// 該当部分のコードを抜粋して貼る
```

**何をしているか：**
スワイプが成功した場合はカードを画面外へ飛ばし、失敗した場合は元の位置へ戻している。
**なぜこう書くのか：**
withAnimationを使うことで、位置の変化を滑らかに表示できる。easeOutは画面外へ飛ぶ動きに適しており、springは元の位置へ自然に戻る動きを表現できる。
**もしこう書かなかったら：**
カードが瞬間移動するように表示され、操作が不自然に感じられる。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API
| 項目                              | 説明                | 使用例                                                     |
| ------------------------------- | ----------------- | ------------------------------------------------------- |
| `DragGesture`                   | ドラッグ操作を検出する       | `.gesture(DragGesture())`                               |
| `onChanged`                     | ドラッグ中の処理を行う       | `.onChanged { value in ... }`                           |
| `onEnded`                       | ドラッグ終了時の処理を行う     | `.onEnded { value in ... }`                             |
| `offset`                        | Viewの位置を移動する      | `.offset(offset)`                                       |
| `rotationEffect`                | Viewを回転させる        | `.rotationEffect(.degrees(rotation))`                   |
| `withAnimation`                 | アニメーション付きで状態を変更する | `withAnimation(.spring) { ... }`                        |
| `@State`                        | 状態を管理し画面を更新する     | `@State private var offset = CGSize.zero`               |
| `DispatchQueue.main.asyncAfter` | 一定時間後に処理を実行する     | `DispatchQueue.main.asyncAfter(deadline: .now() + 0.3)` |


## 自分の実験メモ



**実験1：**
やったこと： swipeThresholdを100から50に変更した。
結果： 少し動かすだけでカードがスワイプされるようになった。
わかったこと： swipeThresholdはスワイプの判定距離を決める重要な値である。

**実験2：**
やったこと： rotation = Double(value.translation.width / 20)をコメントアウトした。
結果： カードが回転しなくなり、左右に動くだけになった。
わかったこと： 回転を付けることで、より自然でリアルな操作感になる。
## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**DragGestureとoffsetはどのような関係がありますか。
   **得られた理解：**DragGestureで取得した移動量をoffsetへ代入することで、指の動きに合わせてViewを移動できる。

2. **質問：**@Stateはなぜ必要ですか。
   **得られた理解：**値が変更されたときに画面を自動で更新するために必要であり、SwiftUIでは状態管理の基本となる。

3. **質問：**withAnimationを使う理由は何ですか。
   **得られた理解：**値の変化を滑らかに表示できるため、ユーザーにとって自然で分かりやすい画面になる。

## この章のまとめ
この章では、SwiftUIのジェスチャー機能を使ってカードをスワイプするアプリを作成した。DragGestureによるドラッグ操作、offsetやrotationEffectによる表示の変化、withAnimationによる滑らかなアニメーション、@Stateによる状態管理を組み合わせることで、実際のアプリのような操作感を実現できることを学んだ。また、複数の機能を組み合わせることで、より使いやすいUIを作成できることを理解した。
（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
